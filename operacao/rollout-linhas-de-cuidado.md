# Rollout — linhas de cuidado (módulo 23, ADR 0040)

Deploy, ligação por cidade e limites conhecidos das linhas de cuidado C2–C7 (api; dashboard, wpda e maintenance seguem
o api). Prova técnica das notas: `pesquisa/2026-10-22-linhas-de-cuidado-prova-tecnica.md`.

## Pré-requisitos

- Cidade em modo registro (`record_mode = record`) com `clinical_record` ligado e utilizável: `care_lines` depende
  dele. Sem ele o interruptor fica "ligado, falta pré-requisito" e as rotas respondem 403 `feature_disabled`.
- Equipes (INE de 10 dígitos, tipo 70/76) com os profissionais (CBO do CNES) nas unidades — módulo 16, retrato do CNES.
  Sem equipe, ninguém recebe sugestão: a pessoa só entra por inscrição manual.
- Bairros e cobertura das unidades (módulo 11): a regra do território (abaixo) só atribui equipe quando o bairro tem
  unidade de referência com UMA equipe ativa.
- Notas metodológicas conferidas (prova técnica acima).
- Nenhuma gem nova.

## Ordem de deploy

Migrações irreversíveis (o `down` levanta `IrreversibleMigration`): não há rollback de schema, só do código
(expand/contract). Publicar a imagem e rodar o `bin/migrate` **dela**, antes de cortar o tráfego. Nunca migrar cidade
fora do rake.

1. **Migração de plataforma** `20261022900001` (`care_line_templates`, com o trigger de imutabilidade em
   `db/platform_triggers.sql`). O `bin/migrate` roda `db:migrate`, `platform:triggers` e `city:migrate:all`, nessa ordem.
2. **Migrações de cidade** (`city:migrate:all`, toda cidade ativa ou suspensa):
   - `20261022900002` — `care_lines`, `care_line_enrollments`, `care_line_suggestions`, `care_outreach_contacts`,
     `citizen_declared_conditions` (triggers em `db/city_triggers.sql`);
   - `20261022900003` — **altera `appointment_requests` do módulo 17**: nova origem `care_line`
     (`origin_care_line_enrollment_id`, um pedido vivo por inscrição), `root_triage_id` passa a aceitar nulo (só o
     pedido da linha e o que nasce de atendimento sem raiz) e os CHECK/triggers de origem mudam. É mudança
     transversal: rode junto os specs do 17 (agendamento, agenda, check-in, drain) e a suíte completa antes do merge.
3. **Catálogo-base**: `bin/rails care_lines:templates:load` UMA vez na plataforma (idempotente; cria a versão 1 das
   seis linhas, nunca sobrescreve).
4. **Protocolo `saude-mental`** publicado no catálogo de triagens (módulo 15). A linha própria de saúde mental e o aviso
   de triagem vencida (repete em 30 dias) dependem dele.
5. **Só depois**, o interruptor `care_lines` (por cidade, pelo maintenance).

Frontends: dashboard e wpda depois do api; depois maintenance e catálogo público.

## Ligar numa cidade

1. maintenance: `care_lines` ("Linhas de cuidado: catálogo C2–C7, inscrição confirmada, busca ativa e prévia dos
   indicadores").
2. Admin municipal (dashboard, com step-up): adotar as linhas do catálogo-base que a cidade vai acompanhar; criar as
   próprias (ex.: saúde mental, com a condição declarada `mental_health`).
3. O job diário (`CareLines::SuggestJob`, fila `housekeeping`, 6h45 America/Sao_Paulo) gera as sugestões; o problema
   registrado na consulta e a condição declarada no wpda geram na hora (consumidores de evento). **Primeira rodada:**
   numa cidade grande, as linhas por idade e sexo (criança, pessoa idosa, mulher) sugerem muita gente de uma vez —
   combine com as equipes adotar essas linhas aos poucos (uma por semana) ou só depois de confirmarem as por problema.
4. Profissionais com vínculo ativo na unidade da equipe: painel e contato. Quem confirma (aceitar sugestão, inscrever à
   mão): médico (CBO 2251/2252/2253) ou enfermeiro (2235). O prefixo `2231` também está em
   `CONFIRM_CBO_PREFIXES`, mas vem da nota (família antiga da CBO 2002): o catálogo de CBO do módulo 10
   (`config/professionals/cbo_saude.yml`) não tem `2231xx` e **não muda**; na prática o médico confirma pelas famílias
   2251/2252/2253. Técnico de enfermagem (322245) vê e registra contato, mas não confirma.
5. Gestão (admin, analista): prévia C2–C7 por equipe, sempre rotulada como estimativa.

## Como o sistema escolhe a equipe da pessoa

- **Inscrita**: a equipe da inscrição.
- **Paciente**: equipe ativa do profissional da consulta finalizada mais recente nos últimos 24 meses **que tenha
  equipe** na unidade do vínculo. Plantonista sem equipe não tira a pessoa da equipe dela.
- **Território (P18)**: quem está há 24 meses sem consulta, ou só com consultas que não dão equipe (plantonista sem
  equipe, membro desligado, equipe inativa, equipe de outra unidade), segue a regra do par validado: a **única** equipe
  ativa das unidades de referência do bairro do par validado mais antigo. Bairro sem unidade, com mais de uma equipe,
  nulo ou sem par validado: **sem equipe, sem sugestão**.
- **Assimetria conhecida**: um paciente sem consulta finalizada em 24 meses também cai na equipe do bairro (via
  `Facts.patients`), mas a sugestão `declared` só nasce para quem tem equipe; e um par validado sem paciente cai pelo
  território (`TeamPopulation.citizen_teams`). Os dois caminhos usam a mesma regra, só partem de pontos diferentes.
- A sugestão nasce já atribuída a uma equipe (`care_line_suggestions.team_id`). Equipe inativa ou sem ninguém no
  território não recebe nada.
- Cidadão **apagado pela exclusão LGPD** (ou rebaixado de "validado") não recebe sugestão e não decide nem pesa no
  aceite de outro par do mesmo CPF.

## "Agendar" e o módulo 17

- "Agendar" na inscrição abre o pedido do 17 com origem `care_line` (201 criado, 200 reaproveitado). A resposta traz
  `can_book`: só quem tem `citizen_verifier` (recepção) pode marcar pelo painel de marcação.
- Encerrar a inscrição por `death` ou `moved` **fecha o pedido da linha** (o vivo, criado pela própria inscrição; o
  reaproveitado de outro tipo nunca) como `dismissed` com texto fixo, e cancela o horário vivo dele.
- **Limitação do 17**: o horário cancelado pela saída por óbito ou mudança aparece como "cancelado pelo cidadão"
  (`cancelled_by_citizen`, como na exclusão LGPD), inclusive no wpda e na barra do analytics de qualidade.
- **Campanhas (módulo 12)**: `Campaigns::Criteria::AppointmentRequestOpen` sem filtro de tipo (`kinds`) passa a incluir
  quem tem pedido `care_line` aberto — a pessoa aguarda agendamento. O público dessas campanhas pode crescer; quem não
  quer isso filtra por `kinds`.

## O que não entra (comunicar às equipes)

- Visita do ACS (módulo 24), vacinas (módulo 29) e saúde bucal (módulo 28): "não acompanhado pelo sistema"; não contam
  como atraso, os pesos saem da prévia e os restantes são redistribuídos.
- Registros feitos fora do Rota Saúde (PEC de outra unidade, outro município): o SIAPS conta, a prévia não.
- Vínculo pela NT 30/2025: a prévia atribui a pessoa pela última consulta (24 meses) ou pelo bairro, não pelo cadastro
  do SIAPS.
- **Gestação (C3), exclusão por aborto (P14)**: o problema excludente (W82/W83/O02–O06/Z303) só vale se o registro
  (último evento `added`/`reactivated`) é da **gestação atual** — a partir da DUM ou, sem DUM, nos últimos 294 dias.
  Registro antigo esquecido como ativo não exclui; registro sem data também não exclui.

## Prévia C2–C7 — como ler

Sempre `estimate: true`; a nota oficial é do SIAPS.

- **Só coortes com a janela fechada**: em prática `end_of_window`, quem ainda não cumpriu enquanto o prazo está aberto
  fica fora do denominador até fechar ou cumprir.
- **Prática sem elegível** (`no_eligible`): sai da conta e o peso dela é redistribuído entre as demais rastreadas.
- **Indicador sem nenhum denominador**: aparece "sem dados" (`score` e `concept` nulos), nunca 0.
- Cada peso é arredondado sozinho (uma casa), por isso a **soma dos pesos não é exibida** e pode não dar exatamente 100.
- `excluded` lista as famílias não rastreadas (`acs_visit`, `vaccine`, `oral_health`); `excluded_practices` traz as
  práticas retiradas da conta.

## Revisão anual das notas (risco do spec §11)

Quando o MS publicar nova nota metodológica C2–C7 (ou NT de pesos), refaça a prova técnica (download, SHA-256,
comparação prática a prática) e grave a versão nova pelo maintenance (`upsertCareLineTemplate`). As cidades continuam
na versão adotada até readotar (`PATCH /care_lines/admin/lines/:id { template_version }`).

## Verificação depois de ligar

- Uma sugestão por problema ativo e a confirmação (paciente de teste); o painel da equipe com a busca ativa; "Agendar"
  com um usuário `citizen_verifier`.
- Em dev (semente de Curitiba, UBS Jardim das Flores): a lista de sugestões da equipe traz 4 ou 5 pendentes (a 5ª, da
  Luíza, só aparece se o worker rodou o job). A equipe e o INE vêm de `GET /care_lines/teams`; o INE da semente é
  derivado (10 dígitos), não fixo.
- Nenhuma nota de contato, condição declarada ou DUM no log do dia (`grep` dos textos de teste).

## LGPD

- Inscrição e contatos **retêm** o pedido de exclusão (são documento clínico). A condição declarada (que não retém) e
  as sugestões do par são apagadas na confirmação da exclusão, antes de o cidadão ser apagado ou anonimizado.
- Nome, CPF, CNS, nascimento, DUM, condição declarada, nota de contato e motivo de recusa/encerramento nunca vão para
  log, evento ou Analytics; os eventos do módulo levam só ids.

## Desligar

- `care_lines` desligado: rotas `/care_lines` e `/citizen/{declared_conditions,care}` respondem 403; o job diário e os
  consumidores param; o catálogo do cidadão continua com o aviso de triagem vencida (é do módulo 15, sem interruptor).
  Os registros ficam (inscrições, contatos, declarações, pedidos `care_line` do 17).
- Migrações irreversíveis: sem rollback de schema; volte só o código (expand/contract).
