# Rollout — vigilância epidemiológica (módulo 25, ADR 0043)

Deploy, ligação por cidade, operação da lista e limites conhecidos da notificação compulsória (api; dashboard e
maintenance seguem o api; o wpda não muda). Spec: `superpowers/specs/2026-10-27-module-25-disease-surveillance-design.md`.
Contrato: `superpowers/plans/2026-10-27-module-25-disease-surveillance-contracts.md` (§14 ficha, §15 K1–K14).
Interruptor: `surveillance` ("Vigilância epidemiológica"). Papel: `surveillance`.

## Pré-requisitos

- **`CityProfile.ibge_code` e `CityProfile.uf` preenchidos.** O interruptor `surveillance` não exige nenhum dos dois
  (o maintenance não mostra falta), mas o município de residência da ficha e a regra "morador de outra cidade"
  (`residence_ibge` ≠ cidade) dependem do IBGE da cidade; sem ele a regra é pulada. Conferir staging e produção.
- **Unidades com CNES.** A notificação grava o CNES da unidade; sem ele a ficha e o CSV saem sem o código.
- **Unidades notificadoras.** A migração marca `notifying = true` nas unidades `kind = 'ubs'` ativas, e a UBS criada
  depois nasce notificadora (inclusive pela importação do CNES) quando `notifying` não é informado; unidade de outro
  tipo o `municipal_admin` liga à mão em Unidades. A negativa semanal e o lembrete só olham unidade notificadora ativa.
- **Quem terá o papel `surveillance`.** Papel privilegiado: concedido pelo `municipal_admin` **com step-up**; o
  mantenedor não concede. Quem tem o papel vê nome do paciente e doença, então a escolha é da secretaria. O papel
  `analyst` vê só o painel (agregado suprimido, sem nome). Os alertas por e-mail vão para **todo membro ativo** com o
  papel `surveillance` (usuário não desativado).
- **E-mail de envio configurado** no api e no worker: `SMTP_ADDRESS`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`,
  `MAIL_FROM`. Sem SMTP o alerta de imediata não sai (o painel e a fila continuam mostrando o caso). Lembrar da
  lição "env do arquivo ≠ env do processo": confirmar com `printenv` no container que está rodando.
- **Worker na imagem nova.** Os consumidores novos (`Surveillance::SafetyNetJob`, `Surveillance::AlertJob`,
  `Surveillance::MailJob`) e os dois jobs recorrentes só existem na imagem nova; worker antigo falha com
  `ActiveJob::UnknownJobClassError`.
- Nenhuma gem nova (o diff do `Gemfile` contra `origin/main` é vazio).

## Ordem de deploy

Migração de **plataforma** (`20261027900001_create_notifiable_diseases`) **e** de **cidade**
(`20261027900002_add_disease_surveillance`) **irreversíveis** (o `down` das duas levanta `IrreversibleMigration`): não
há rollback de schema, só do código (expand/contract). Publicar a imagem e rodar o `bin/migrate` **dela**, antes de
cortar o tráfego. Nunca migrar cidade fora do rake.

1. **api**: publicar a imagem (api e worker); rodar o `bin/migrate` da imagem (`db:migrate`, `platform:triggers`,
   `city:migrate:all`).

   ```bash
   bin/migrate
   ```

   - Plataforma: cria `notifiable_disease_versions` e `notifiable_diseases` (lista versionada; nenhuma versão nem
     doença muda ou some) e os triggers de imutabilidade.
   - Cidade: o CHECK `ck_memberships_role` passa a aceitar `surveillance`; `health_units.notifying`
     (padrão `false`, `true` nas UBS ativas existentes); `local_notifiable_diseases`, `disease_notifications`,
     `disease_notification_events`, `weekly_negative_reports`; índice parcial de no máximo uma notificação aberta por
     paciente e doença (`idx_disease_notifications_one_open`); triggers de guarda (`db/city_triggers.sql`).

2. **Carga da lista nacional, uma vez, na plataforma (OBRIGATÓRIO).** Sem lista não há sugestão, rede de segurança
   nem notificação (a lista vigente é a base de tudo). Idempotente: só cria a versão 1 com a tabela vazia e **nunca
   sobrescreve** a curadoria feita no maintenance.

   ```bash
   bin/rails surveillance:load_disease_list
   ```

   Saída esperada na primeira vez: `versões criadas: 1`; nas seguintes, `0`. A carga é a Portaria GM/MS 11.211/2026
   (71 doenças, fonte por linha em `config/surveillance/notifiable-diseases.yml`).

3. **dashboard** (depois do api).
4. **maintenance** (depois do dashboard): rótulo do interruptor e do módulo no catálogo e a tela da lista de doenças.
5. **Só depois**, ligar o interruptor `surveillance`, **por cidade**, no maintenance. A sugestão de notificação na
   consulta **só aparece com `clinical_record` também ligado**; sem ele, com `surveillance` ligado, as rotas da
   consulta respondem 403 `feature: "clinical_record"` e a vigilância trabalha por notificação manual (cidade sem
   modo prontuário, hospital, laboratório, outra cidade).
6. Conceder o papel `surveillance` às pessoas da vigilância (Equipe, com step-up).

## Conferências pós-deploy

- Lista carregada e vigente, na plataforma:

  ```sql
  SELECT version, effective_from FROM notifiable_disease_versions ORDER BY version;
  SELECT count(*) FROM notifiable_diseases;   -- 71 na versão 1
  ```

- Triggers presentes na plataforma (esperados: `notifiable_disease_versions_immutable`, `notifiable_diseases_immutable`):

  ```sql
  SELECT tgname FROM pg_trigger WHERE tgname LIKE 'notifiable_disease%';
  ```

- Triggers presentes em **cada banco de cidade** (esperados: `disease_notifications_guard`,
  `disease_notification_events_reencryption_only`, `weekly_negative_reports_append_only`,
  `local_notifiable_diseases_no_delete`, mais os `*_append_only_truncate` das quatro tabelas):

  ```sql
  SELECT tgname FROM pg_trigger
   WHERE tgname LIKE '%disease_notification%' OR tgname LIKE '%weekly_negative%'
      OR tgname LIKE '%local_notifiable%';
  ```

- Unidades notificadoras = UBS ativas (e só elas, até o admin ligar outras):

  ```sql
  SELECT kind, active, notifying, count(*) FROM health_units GROUP BY 1, 2, 3 ORDER BY 1, 2, 3;
  ```

- Jobs recorrentes registrados no worker: `surveillance_overdue` (todo dia, 7h15 de São Paulo) e
  `surveillance_negatives` (segundas, 7h30).
- **E-mail de teste de imediata em staging**: com `surveillance` ligado, criar uma notificação de doença imediata e
  conferir que o e-mail chega aos membros `surveillance`, com assunto "[rota-saúde] Nova notificação imediata:
  <doença>, <unidade>", **sem nome, CPF, CNS, bairro, data de sintomas nem id/link com id**.
- `GET /surveillance/diseases` devolve a lista vigente (71) para `surveillance` e `health_professional`.

## Operação

- **A lista muda por portaria.** Não é deploy: um mantenedor com step-up publica uma **versão nova** da lista no
  maintenance (a lista inteira a cada publicação, com portaria, `expectedVersion` e **data de vigência**). A vigência
  não começa antes de hoje nem antes da vigente; data futura fica "agendada". Versão nova pode retirar doença. A
  versão antiga fica (a notificação grava a versão da lista com que nasceu). Cada publicação gera auditoria de
  plataforma (`maintenance.notifiable_disease_list.published`); `stale_version` significa que outra pessoa publicou
  antes: recarregar e refazer. A carga inicial (`load_disease_list`) **não** serve para atualizar: não toca em lista
  existente.
- **Caxumba aguarda retificação.** O art. 2º da Portaria diz imediata, o Anexo I diz semanal; a lista segue o anexo
  (semanal), com a nota na doença. Se o Ministério retificar, publicar versão nova.
- **Doenças estaduais = doenças locais da cidade.** A vigilância cria/edita/desativa em
  `/surveillance/admin/local_diseases` (escrita com step-up): código `local_<a-z0-9_>`; código ou CID/CIAP em
  colisão com a plataforma vigente ou com outra local → 409 `duplicate_code`. Doença local nunca é apagada (trigger);
  desativa-se. Se uma versão nova da plataforma passar a cobrir um CID/CIAP de uma local, a **plataforma vence** na
  correspondência e a local só é recusada na próxima edição dela.
- **Unidade notificadora**: o admin liga/desliga em Unidades (`notifying`). A negativa semanal é registrada pela
  vigilância em nome da unidade; unidade com caso na semana não registra negativa (409 `unit_has_notifications`).
- **Fila**: imediatas e vencidas primeiro. Vencida = prazo passado com situação `possible` ou `notified`, contado do
  instante da suspeita. Transições lineares (`possible → notified|dismissed`, `notified → investigating →
  closed → entered`); `entered` exige o número do Sinan; `entered` e `dismissed` são terminais.
- **Rede de segurança**: consulta finalizada (ou adendo) com CID/CIAP da lista, problema novo ou reativado e sem
  notificação aberta ou recente da doença gera "possível notificação não feita" na fila, sem bloquear nada.
- **Alertas**: imediata criada → e-mail a todos os membros `surveillance`; resumo diário das vencidas (só
  contagens); lembrete semanal de unidades sem negativa nem caso na semana anterior (só nomes de unidade).

## O que o módulo NÃO faz

- **Não envia nada ao Sinan, e-SUS Sinan ou e-SUS Notifica** (não há interface para terceiros). A vigilância
  **digita à mão** no sistema oficial, usando a ficha impressa em PDF, e **volta ao Rota Saúde para informar o número
  oficial** (transição `closed → entered`). Notificação encerrada sem número aparece no relatório "sem número oficial".
- Não gera as fichas de investigação específicas por doença: a v1 imprime a ficha de notificação individual comum.
- Não faz detecção espaço-temporal nem declara surto: o painel mostra **sinal para avaliar**, nunca surto.
- Não registra vacina, declaração de óbito ou de nascido vivo, nem o GAL; o cidadão não vê a notificação no wpda.
- Doença imediata sem CID próprio (óbito por dengue, Zika em gestante, violência sexual, tentativa de suicídio etc.)
  e as 19 doenças sem CID (`cid10: []`) **só entram por notificação manual**; SRAG também (o J11 da ficha é o da
  gripe sazonal).

## LGPD

- **A notificação é obrigação legal e retém o cadastro do paciente**: qualquer notificação, em **qualquer situação**
  (`dismissed` inclusive), retém o pedido de exclusão (`Citizens::RequestErasure`). Nada é apagado nem anonimizado por
  esse pedido; informar o titular de que a retenção é por obrigação legal.
- Ficha complementar e motivo do andamento são cifrados. Nome, CPF, CNS, doença e sintomas nunca vão para log, evento,
  e-mail ou URL; os eventos (`disease_notification.created/.updated/.transitioned/.dismissed`,
  `weekly_negative.recorded`, `local_disease.changed`) levam só ids e booleanos, nunca `disease_code`.
- Leitura nominal (lista, detalhe, PDF, CSV, busca de pessoa) grava `clinical_record.viewed` com
  `access: "surveillance"`; contagens e o painel não gravam trilha. O CSV sai sem CPF e com o CNS pelos 4 últimos
  dígitos; o `Content-Disposition` não leva dado pessoal.
- O e-mail só leva doença e unidade (imediata), contagens (vencidas) e nomes de unidade (negativa).

## Desligar e reverter

- Desligar o interruptor `surveillance` (maintenance): rotas respondem 403 `feature_disabled`; nenhuma sugestão, fila,
  e-mail nem negativa nova; **nada se apaga** (notificações, andamento, negativas, doenças locais e versões da lista
  ficam). Os consumidores (`SafetyNetJob`, `AlertJob`) e os jobs diário/semanal ficam **inertes** com o interruptor
  desligado.
- Ligar de novo **não reprocessa** nada do intervalo desligado (ver limites conhecidos).
- As migrações **não voltam**: sem rollback de schema; volte só o código (expand/contract). O papel `surveillance`
  segue aceito no CHECK e a coluna `health_units.notifying` segue existindo.
- Lista errada publicada: publicar outra versão (a anterior não é apagada nem editada).

## Limites conhecidos e riscos aceitos

1. **K14 — sinal do painel e supressão.** O sinal usa os valores **publicáveis** como linha de base: célula 0 ou ≥ 5
   entra crua; célula suprimida (1–4) entra como 2,5. Assim o sinal depende só do que o cliente já pode ver. Mas
   diferenciar o painel sem filtro contra um painel por doença só produz **faixas**, não o valor exato. E um bairro
   novo com 5 casos já é sinal (base 0). O sinal é sempre "para avaliar".
2. **"Carregar mais" na fila pode pular uma linha** que fica vencida entre duas páginas (cursor com 4 valores). A linha
   aparece no contador de vencidas e na aba "Vencidas"; recarregar a fila resolve.
3. **Ligar o interruptor depois não reprocessa** consultas finalizadas enquanto ele estava desligado: não há rede de
   segurança retroativa. A vigilância cobre esse intervalo por notificação manual.
4. **E-mail.** Falha transitória de SMTP faz o job repetir (até 5 tentativas) e **pode reenviar a quem já recebeu**
   (duplicata aceita: melhor repetir que perder). Falha permanente de um endereço (`SMTPFatalError`/`SMTPSyntaxError`)
   é registrada só como contagem (log `warn`, nunca o endereço) e os demais recebem. Sem destinatário: log com 0, sem
   erro.
5. **Rede de segurança usa a lista vigente quando o job roda**, não a da data da suspeita. Erro que não seja
   deadlock (que tem retry) vira **execução falha** no Solid Queue e precisa de retry manual; o evento é desfeito
   (`ProcessedEvent`) e o retry reprocessa. Olhar execuções falhas de `Surveillance::SafetyNetJob` no pós-deploy.
6. **PDF impresso também para `possible` e `dismissed`, sem linha de situação** (decisão pendente, a levar à
   coordenação): há risco de a vigilância digitar no Sinan uma notificação descartada. Até decidir, orientar a só
   imprimir a partir de `notified` em diante. Proposta: linha de situação em "Dados gerais".
7. **Doença local desativada não pode ser usada como filtro do painel** (o filtro só aceita doença ativa); os casos
   antigos dela continuam nas contagens sem filtro.
8. Ficha pré-preenchida **omite** (deixa vazio) valor do paciente que não cabe no limite ou formato da ficha (rua 60,
   número 6, complemento 30), em vez de cortar: a vigilância digita.
9. Morador de outra cidade fica na cidade do atendimento, marcado "residente em outro município"; é digitado no Sinan
   daqui, que cuida da transferência. Nenhum fluxo entre bancos na v1.

## Gates de go-live (vigilância da cidade piloto)

- **Lista conferida pela vigilância da cidade piloto, incluindo `repeat_after_days` (K3, K15)**: tuberculose 270,
  hanseníase 365, sífilis adquirida e gestante 90, sífilis congênita 36500 (uma por criança), HIV em
  gestante/criança exposta 270 (uma por gestação), HIV/AIDS e hepatites virais 36500 (nunca repetir), demais 30.
  Os valores de sífilis congênita e de HIV em gestante/criança exposta são proposta do contrato (K15) e precisam da
  validação da vigilância piloto.
- **Códigos genéricos (K13)**: validar os **manuais** (F99, L98.9, C80, Z20.9) e os **mantidos** (W64, Y09, O98.6,
  A92.8, Z57.9) — estes seguem disparando sugestão.
- **SRAG só manual** (K13a); **malária imediata** (região extra-amazônica; a nota cobre a região amazônica).
- **P35.4 ausente** da tabela CID-10 do DATASUS 2008; **Oropouche** apoiado na NT 117/2024 (cópia da SES-MG);
  **resposta do Fala.BR sobre a caxumba não vendorizada** (só citada na nota da doença).
- **Tuberculose**: a carga não traz o CIAP-2 A70 (cobre sequelas) e o K13e inclui os extrapulmonares do GVS que não
  colidem; confirmar o conjunto final de CID com a vigilância (o enfermeiro, que registra CIAP-2, notifica TB à mão).
- **Ficha impressa conferida contra a digitação no Sinan** (campos 12–30; `form.<chave>` do §14), com um caso real.
- **E-mail de teste em staging chega sem dado do paciente** (ver conferências).
- `CityProfile` com IBGE e UF; unidades com CNES; unidades notificadoras revisadas.
- **Quem recebe o papel `surveillance`** definido e concedido pelo admin com step-up.
- Decisão pendente do PDF de `possible`/`dismissed` (limite conhecido 6) resolvida ou orientação passada à equipe.
