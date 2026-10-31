# Rollout — saúde bucal (módulo 28, ADR 0046)

Deploy, ligação por cidade, implantação (unidades CEO, equipes eSB, lista do TSB/ASB, exame inicial), operação e
limites conhecidos do atendimento odontológico da APS e do CEO (api; dashboard e maintenance seguem o api). Spec:
`superpowers/specs/2026-10-30-module-28-oral-health-design.md`. Contrato: §16 (decisões O1–O36).
Interruptor: `dental_care` ("Saúde bucal"), que **requer `clinical_record`**. Rota nova sob `/dental`; a consulta
segue sob `/attendance` (tipo `dental` da consulta do 19a).

## Pré-requisitos

- **Prontuário ligado na cidade** (`clinical_record`; módulo 16). Sem ele o `dental_care` não fica utilizável
  (o maintenance aceita ligar, mas fica "ligada, falta pré-requisito"; o api responde 403 `feature_disabled` com
  `feature: "clinical_record"`). O modo de prontuário e a exportação LEDI estão em
  [`rollout-modo-de-prontuario.md`](rollout-modo-de-prontuario.md) e [`rollout-prontuario.md`](rollout-prontuario.md).
- **Profissionais com vínculo profissional ativo e CBO do CNES** (módulo 10): dentistas `2232*` (todas as
  especialidades), TSB (`322405`, `322425`) e ASB (`322415`, `322430`). Sem o CBO o vínculo não dá o papel
  odontológico (`dental_role` vem `null` em `consultation_options`) e a consulta odontológica não abre.
- **CNES das unidades CEO** conferido e a unidade ativa no módulo 09. O CEO é operado pela cidade; CEO de consórcio
  ou de outro município segue como prestador externo do 26a, sem prontuário, FAO nem metas aqui.
- **Equipes de saúde bucal (eSB) no CNES da cidade**, com INE próprio, para a importação.
- **Perfil profissional com nome** para toda a equipe (dentista, TSB, ASB): o nome sai do perfil.
- **Worker na imagem nova** (jobs e consumidores de eventos do módulo; worker antigo falha com
  `ActiveJob::UnknownJobClassError`).
- Nenhuma gem nova; nenhuma migração de plataforma.

## Ordem de deploy

Migração de **cidade** `20261030900001` **irreversível** (sem rollback de schema, só do código: expand/contract).
Ela:

- troca os CHECKs de `consultations` (tipo `kind`, `baseline`, tipo de atendimento condicional), de `health_units`
  (`ck_health_units_kind` passa a aceitar `ceo`) e de `health_teams` (`ck_health_teams_kind` passa a aceitar `71` e
  entra `ck_health_teams_associated_team`);
- cria as tabelas do módulo (odontograma por eventos, planos e itens, periograma, registro periodontal,
  endodontia, pedidos de prótese, perfis do CEO) e as guardas de imutabilidade;
- cria o índice `idx_consultations_kind_status_finalized` (`kind, status, finalized_at`) **sem `CONCURRENTLY`**:
  enquanto ele é construído, **`consultations` fica travada para escrita** (finalizar, rascunho, adendo). Meça o
  tempo em staging com volume real e **agende fora do pico**, com aviso às unidades. Os CHECKs também varrem a
  tabela (`consultations`, `health_units`, `health_teams`) ao serem recriados.

Publicar a imagem e rodar o `bin/migrate` **dela**. Nunca migrar cidade fora do rake.

1. **api**: publicar a imagem (api e worker) e rodar o `bin/migrate` da imagem (`db:migrate`, `platform:triggers`,
   `city:migrate:all`), na janela combinada.

   ```bash
   bin/migrate
   ```

2. **dashboard** (depois do api).
3. **maintenance** (depois do dashboard): rótulo do interruptor e do módulo no catálogo.
4. **Ligar e preparar, nesta ordem:**
   1. maintenance: ligar `dental_care` na cidade (com `clinical_record` já ligado).
   2. Cadastrar as **unidades CEO** (seção abaixo).
   3. Importar o CNES, conferir as **eSB** e as associações (seção abaixo).
   4. Conferir a **lista de procedimentos do TSB/ASB** (seção abaixo).
   5. **Exame inicial** dos pacientes em tratamento (seção abaixo).
   6. **Só então anunciar** o atendimento odontológico às unidades.

## Unidades CEO

- Admin municipal: abrir a unidade em `unidades de saúde`, trocar o tipo para `ceo` e preencher o **perfil do CEO**
  (`ceo_profiles`): **tipo I, II ou III** e as **especialidades** que oferece (endodontia, periodontia, cirurgia
  oral menor, estomatologia, pacientes com necessidades especiais, prótese). O tipo define a meta mensal por
  especialidade (Portaria 1.464/2011, Anexo XL da PC 6/2017).
- Unidade com perfil de CEO não volta para outro tipo (409 `unit_has_ceo_profile`); metas e perfil de unidade que
  não é CEO respondem `unit_not_ceo`.
- O CNES da unidade é o que vai na FAO: a produção do CEO sai pela mesma ficha, no CNES do CEO.

## Equipes eSB e associação à eSF/eAP

- As eSB (tipo `71`) vêm da **proposta do CNES** (importação do módulo 24/10), com INE próprio; dentista, TSB e ASB
  se vinculam a elas como já se faz com eSF e eAP. O especialista do CEO **não** tem equipe e não entra no B1.
- **Associação eSB → eSF/eAP:** automática quando a unidade tem **uma só** eSF/eAP ativa. Nos outros casos, o admin
  associa em `GET/PUT /dental/admin/oral_health_teams` (**step-up**; evento `dental.team_association_changed`).
  Sem associação o indicador B1/B2 sai com denominador nulo ("—") e numerador cru.
- Linhas de cuidado e território continuam só com `70`/`76`.

## Lista do TSB e do ASB

A cidade define a lista de procedimentos SIGTAP que o TSB e o ASB podem registrar (envelope
`{ "procedures": [ { "sigtap", "name" } ] }` em `GET/PUT /dental/admin/auxiliary_procedures`, `municipal_admin` com step-up). Eles registram a consulta odontológica com
procedimentos dessa lista, que geram eventos no odontograma e dão baixa em item do plano, mas **não** criam nem
cancelam item, **não** encaminham, **não** emitem documento e **não** abrem plano. Lista vazia = o auxiliar não
consegue registrar nada. Aprovar a lista com o responsável técnico da saúde bucal antes de ligar.

## Exame inicial (`baseline`) dos pacientes em tratamento

Quem já está em tratamento entra no sistema por uma **consulta odontológica marcada como registro inicial**
(`baseline`): só a situação dos dentes (achados), **sem** procedimento, tipo de consulta nem plano, e **não gera
FAO**. Exige achados (`baseline_requires_findings`). É trabalho de implantação: o dentista faz uma por paciente,
antes do primeiro atendimento real, na unidade onde o paciente é seguido. Atendimento real continua sendo consulta
normal (com FAO). Sem o exame inicial o odontograma nasce vazio e o plano sugere do zero.

## Laboratório de prótese

- Laboratório **próprio ou conveniado** do 26a: cadastrado como prestador (`care_providers`); o dentista escolhe em
  `GET /dental/laboratories`.
- Laboratório **terceiro** (sem cadastro no 26a): informado **por nome** (3 a 120 caracteres) e cadastrado como
  **"Terceiro" no SCNES da unidade que entrega** a prótese. A produção conta **na instalação**, registrada pelo
  dentista na consulta, no CNES da unidade que entrega (NT 32); o protético não usa o sistema.
- Etapas: moldagem, enviada, recebida, instalada. `installed` só pela consulta; transição fora de ordem é 409
  `invalid_transition`.

## Prova no PEC e novos motivos de falha LEDI

A FAO sai pelo exportador LEDI do módulo 16 (agregação por SIGTAP somando a quantidade; limites: dente ≤ 32,
sextante ≤ 6, arcada ≤ 2, demais ≤ 1; até 20 códigos por ficha). Três pontos (O19) são conferidos no PEC local
antes do go-live e, se o PEC recusar, o construtor da ficha é corrigido e o fato anotado aqui:

- **CID-10 do dentista** aceito na FAO (se recusado: só CIAP-2 e avisar a coordenação);
- **`gestante` omitido** para paciente do sexo masculino;
- **`localAtendimento = 1`** também no CEO.

Motivos novos em `ledi_generation_failures`, por rascunho que escapou da finalização:
`too_many_procedure_codes` (mais de 20 códigos distintos) e `quantity_exceeds_location_limit` (soma acima do limite
do local). A finalização já recusa os dois; a falha só aparece se o dado chegou por outro caminho. **Esses motivos
ainda não têm rótulo no dashboard nem no maintenance** (aparecem com o código cru); conferir onde os motivos do
exportador são exibidos. Tratamento: a ficha **nunca trunca** procedimento; corrigir o atendimento por adendo e
reenfileirar.

## O que o módulo 28 não faz

- **Não gera BPA do CEO** para o pagamento MAC: a NT 32 diz que o BPA deixou de valer para os serviços habilitados,
  mas não diz se a produção de média complexidade (PPI/FPO) ainda o exige. **Pendente**; se exigido, reaproveita o
  módulo 22.
- Sem escovação supervisionada e demais ações coletivas (indicador B4), imagem radiográfica, ortodontia, implante e
  Sesb (Portaria 751/2023).
- Indicadores **B3, B5 e B6** e o **Quadro 3 da NT 32** (lista oficial de próteses) **não lidos na fonte oficial**:
  pendentes.
- B1 (primeira consulta programática) e B2 (tratamento concluído) calculados por eSB; `period=YYYY-MM` = os 12
  meses que terminam nesse mês.

## LGPD

- Odontograma, plano, periograma, endodontia e prótese seguem a regra de leitura do prontuário (ADR 0031: em
  contexto ou abertura justificada); cada leitura nominal deixa trilha `clinical_record.viewed`. Painéis de metas e
  indicadores só com contagens, sem dado pessoal e sem trilha.
- Dente, face, situação e motivo **não vão a evento, log, URL, `inspect` ou `as_json`**; o histórico do dente
  pagina por `before`/`limit` (sem `?tooth=`).
- A exclusão LGPD **retém tudo**, como o prontuário (ADR 0026).
- Linha do tempo: a entrada continua `consultation`, com título "Consulta odontológica" ou "Exame odontológico
  inicial" (inclusive para o cidadão), sem resumo.

## Desligar e reverter

- Desligar `dental_care` (maintenance), decisões O16/O33: **nenhuma escrita odontológica nova** (rascunho, plano,
  prótese, perfil, adendo com `changes.dental` respondem 403 `feature_disabled`). As consultas **já finalizadas
  continuam legíveis**, com a linha do tempo e o vocabulário (`dental` em `consultation_options` vai sempre).
  Consulta comum não é afetada. Nada se apaga; ligar de novo não reprocessa nada.
- Rascunhos odontológicos abertos ao desligar ficam parados: finalize ou descarte antes.
- A migração **não volta**: sem rollback de schema; volte só o código (expand/contract). Os CHECKs novos, o tipo
  `ceo` e a eSB `71` seguem aceitos.
- Desligar `clinical_record` com `dental_care` ligado **é permitido** (`Platform::Features.set!` não confere
  pré-requisito): o `dental_care` fica "ligada, falta pré-requisito" (`clinical_record_disabled` em `missing`),
  inutilizável, e todo `/dental/*` responde 403 `feature_disabled` com `feature: "clinical_record"`. Ligar o
  `dental_care` sem o prontuário também é aceito, com o mesmo efeito.

## Gates de go-live

- **Código `71` da eSB confirmado.** Está só em fonte secundária (ficha do CNES de Campinas; NT 23 do COSEMS-SP),
  não no manual do SCNES. Confirmar com o CNES da cidade.
- **Códigos sucessores de restauração nas metas do CEO confirmados** com a coordenação de saúde bucal. A portaria
  lista `0307010023/31/40`; a SIGTAP de 06/2026 traz as restaurações por material e dentição (`0307010082` a
  `0307010139`) e o TRA (`0307010074`). O YAML os inclui no grupo básico e na regra dos 50% como **inferência**
  (`source: successor`).
- **BPA do CEO para o MAC**: decidir com a secretaria se é exigido (pendente).
- **Quadro 3 da NT 32** (próteses) e **notas metodológicas B3–B6** lidos na fonte oficial.
- **Prova no PEC** das três verificações do O19 (acima), com as cinco fichas aceitas e a quantidade somada correta.
- **Lista do TSB/ASB** aprovada; **tipo e especialidades de cada CEO** conferidos; **associação das eSB** conferida.
- Tempo da migração medido em staging (índice não concorrente em `consultations`) e janela combinada.
- Dev: lista de situações do odontograma (21 de coroa, 6 de raiz, prótese total por arcada) conferida na versão do
  PEC local.

## Dev: contas da semente

Só em dev (a semente não roda em ambiente publicado), em Curitiba, com `dental_care` ligado, o CEO tipo II
(`CEO Curitiba Centro`), a eSB `eSB Jardim das Flores` associada à eSF e o laboratório
`Laboratório de Prótese Dental Sorriso Ltda`. Senha e TOTP: ver a semente (não repetidos aqui).

| Papel | Nome | E-mail | CBO |
|---|---|---|---|
| Dentista da eSB | Fernanda Cristina Moreira | `dentista@curitiba.demo` | 223208 |
| Auxiliar (ASB) | Juliana Aparecida Santos | `asb@curitiba.demo` | 322430 |
| Endodontista do CEO | Rodrigo Henrique Tavares | `endodontista@curitiba.demo` | 223212 |

O mantenedor de dev é o da semente do maintenance (`dev@local`). A semente é idempotente e fictícia (pacientes das
sementes de linhas de cuidado e regulação).

## Riscos conhecidos

- **Migração trava `consultations`** durante o índice: atendimento parado na cidade pela duração da construção.
- **Eventos do odontograma são imutáveis:** erro de marcação só se corrige por adendo/novo evento.
- **PEC pode somar dentes de forma diferente** da nossa agregação (a prova no PEC cobre).
- **Sem BPA do CEO:** se o MAC exigir, a produção do CEO só existe na FAO até o módulo 22 ser estendido.
- **Dev:** a migração de cidade é irreversível e deixa os bancos de dev à frente da main (a parity spec pode
  quebrar em main até o DROP + `city:test_databases`); apague os bancos `_mod28` no fechamento.
