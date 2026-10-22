# Módulo 23 — Linhas de cuidado — design

**Data:** 2026-10-22
**Status:** aprovado (2026-10-22)
**Afeta:**
- `apps/api`: plataforma — catálogo-base das linhas (C2–C7) com boas práticas e regras de elegibilidade; cidade — `care_lines` (adoção e linhas próprias), `care_line_enrollments`, `care_line_suggestions`, `care_outreach_contacts`, `citizen_declared_conditions`; avaliação das boas práticas na leitura; interruptor `care_lines`; aviso de triagem periódica vencida (pendência do módulo 15).
- `apps/dashboard`: catálogo de linhas da cidade; sugestões a confirmar; inscrição e saída; painel nominal da equipe com busca ativa; registro de contato; prévia de indicadores por equipe.
- `apps/wpda`: condição de saúde declarada no perfil; "Meu acompanhamento" (o que está para fazer e triagem periódica vencida).
- `apps/maintenance`: interruptor (mecanismo genérico) e catálogo-base.

**ADR:** `docs/adr/0040.md` · **Módulo:** `docs/modulos/23--linhas-de-cuidado.md` · **Pesquisa:** `docs/pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md` (§5 cofinanciamento, §6 indicadores C1–C7)

**Fora desta entrega:** visita do ACS (módulo 24) e vacinas (módulo 29) como fontes — aparecem como "não acompanhado pelo sistema"; envio de indicador ao SIAPS (o oficial é do SIAPS); mensagens automáticas (SMS/e-mail) de busca ativa; modo integrado.

## 1. Intenção

Ninguém de uma linha de cuidado some sem a equipe perceber. A equipe vê, pessoa a pessoa, o que está em dia, vencendo ou vencido e age (busca ativa); a gestão vê uma prévia dos indicadores que definem o repasse federal (componente de qualidade, C2–C7) e se antecipa.

**Sucesso:** a equipe abre o painel e sabe quem procurar esta semana e por quê; o cidadão sabe o que está para fazer; a prévia mostra onde a equipe vai perder pontos antes da extração oficial.

**Antiobjetivos:** inscrever alguém numa linha sem um profissional confirmar; contar como atraso o que o sistema não tem como saber (visita do ACS, vacina); apresentar a prévia como valor oficial; expor ao cidadão diagnóstico que ele não conhece.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | A plataforma traz seis linhas prontas, com as boas práticas das notas metodológicas: criança até 2 anos (C2), gestante e puérpera (C3), diabetes (C4), hipertensão (C5), pessoa idosa (C6), mulher e prevenção de câncer (C7). A cidade adota e pode criar linhas próprias (ex.: saúde mental) com metas e periodicidade próprias. |
| 2 | O sistema sugere a inscrição (problema ativo CIAP/CID da lista de problemas do 19a; idade e sexo para criança, idosa e mulher; condição declarada pelo cidadão); um profissional sempre confirma; inscrição manual possível; saída com motivo. |
| 3 | O acompanhamento é calculado na leitura a partir das fontes existentes (consultas 19a, sinais da escuta 18, exames 21, procedimentos 22), sem cópia. Visita do ACS e vacina ficam "não acompanhado pelo sistema" e não contam como atraso. |
| 4 | Painel nominal por equipe e unidade, com filtro de busca ativa, registro de tentativa de contato e atalho para agendar (módulo 17). |
| 5 | "Meu acompanhamento" no wpda, só para par validado, com o que está para fazer e o aviso de triagem periódica vencida (módulo 15). |
| 6 | Prévia de C2–C7 por equipe, marcada como estimativa, nunca como valor oficial. |
| 7 | Só cidades em modo registro (`clinical_record`). |
| 8 | Quem entra: pacientes **e** cidadãos com cadastro validado que nunca consultaram. As sugestões por idade e sexo e por condição declarada também olham os pares validados; ao confirmar a inscrição, o paciente é criado por `Patients::Resolve` (como no 21a). Par só declarado fica fora. |
| 9 | Prévia: denominador = população elegível pelas regras do oficial (problema ativo, idade e sexo) entre pacientes e cidadãos validados da equipe, inscritos ou não; numerador = pontuação oficial por boa prática, com visita do ACS e vacina fora da conta e os pesos restantes redistribuídos (com aviso); problema resolvido tira do denominador e gera **sugestão** de encerramento da inscrição, nunca encerramento automático. |

As decisões 3, 6 e 7 e o recorte de fontes são do coordenador, apresentados ao usuário e aprovados sem discussão de alternativas; as decisões 8 e 9 foram escolhidas pelo usuário depois da revisão da spec.

**Decisões do coordenador (revisão da spec):**
- **Metas por contagem** (ex.: 9 consultas até 2 anos, 7 PA na gestação): ficam `overdue` quando a contagem está abaixo do esperado proporcional ao tempo decorrido da janela (`ceil(min_count × decorrido / janela)`), e `due_soon` quando o próximo marco proporcional vence em 30 dias. O formato das boas práticas ganha `schedule: proportional | end_of_window` (padrão `end_of_window` para contagem 1).
- **Consulta que conta:** só consulta finalizada de médico ou enfermeiro (CBO 225* ou 2235*), como nas notas metodológicas; a `practice` de consulta carrega o filtro de CBO.
- **Acesso:** painel nominal e registro de contato para `health_professional` com vínculo ativo na unidade da equipe; confirmar inscrição só médico ou enfermeiro; prévia para `municipal_admin` e `analyst` (agregado por equipe, sem nome).
- **Condição declarada:** é dado pessoal, não diagnóstico — na exclusão LGPD é apagada e **não** retém o pedido; inscrição e contatos retêm (documento clínico).
- **Linhas próprias:** têm `eligibility` na mesma forma das do catálogo; a condição declarada `mental_health` sugere a linha própria que a cidade marcar para ela.
- **Triagem periódica vencida:** depende do interruptor do módulo 15 (não de `care_lines`); aparece na tela de triagens do wpda para qualquer oferta com intervalo de repetição, inscrito ou não. "Meu acompanhamento" mostra o aviso quando `care_lines` está ligado.
- **DUM:** informada na confirmação da linha de gestante como `anchor_date`.

## 3. Interruptor

- `care_lines` (`requires: ["feature:clinical_record"]`): liga tudo do módulo. Desligado: o wpda não oferece condição declarada nem "Meu acompanhamento"; o dashboard esconde o grupo.

## 4. Dados

**Plataforma (catálogo-base, mantido no maintenance):**
- `care_line_templates`: `key` (`child_c2`, `pregnancy_c3`, `diabetes_c4`, `hypertension_c5`, `elderly_c6`, `women_c7`), `name`, `indicator` (`C2`…`C7`), `eligibility` (regras: códigos CIAP/CID ativos, faixa de idade, sexo, gestação), `practices` (lista: `key`, `label`, `source` — `consultation`, `blood_pressure`, `weight_height`, `exam:<sigtap>` (pedido ou resultado), `procedure:<sigtap>`, `acs_visit`, `vaccine:<código>` —, `window_days`, `min_count`, `schedule` (`proportional` | `end_of_window`), `cbo_filter` (consulta), `anchor` — desde a inscrição, nascimento, DUM ou janela móvel), `version`.
- Versão nova não muda inscrições existentes até a cidade adotar a versão.

**Cidade:**
- `care_lines`: linha adotada (`template_key` + `template_version`) ou própria (`custom`), `name`, `eligibility` e `practices` (as próprias seguem a mesma forma), `declared_condition` (opcional, para a condição declarada sugerir a linha própria), `active`.
- `care_line_enrollments`: `patient_id`, `care_line_id`, `team_id` (equipe de referência), `enrolled_by`, `enrolled_at`, `source` (`suggestion` | `manual`), `anchor_date` (ex.: DUM, nascimento), `status` (`active` | `ended`), `ended_reason` (`discharge` | `death` | `moved` | `pregnancy_ended` | `other`), `ended_at`, `ended_by`. Única ativa por paciente e linha.
- `care_line_suggestions`: `patient_id` ou `citizen_id` (par validado sem paciente), `care_line_id`, `reason` (`problem` | `age_sex` | `declared`), `evidence_ref` (sem texto clínico), `status` (`pending` | `accepted` | `dismissed`), `decided_by`, `decided_at`, `dismiss_reason`. Gerada por job diário e por evento (problema ativado, condição declarada).
- `care_outreach_contacts` (só acréscimo): `enrollment_id`, `channel` (`phone` | `home_visit` | `wpda_message` | `in_person`), `outcome` (`reached` | `no_answer` | `scheduled` | `refused`), `note` (cifrada, opcional), `recorded_by`, `recorded_at`.
- `citizen_declared_conditions` (só acréscimo, cifrado): `citizen_id`, `condition` (`diabetes` | `hypertension` | `pregnancy` | `mental_health` | `none`), `declared_at`, `withdrawn_at`. Declarada pelo próprio cidadão; nunca é diagnóstico.

## 5. Acompanhamento (leitura)

- Para cada inscrição ativa e cada boa prática: a fonte é consultada na janela e devolve `done_at` (última vez), `due_on`, `status` (`ok` | `due_soon` — vence em 30 dias | `overdue` | `untracked` — fonte fora do sistema). Nunca há cópia do dado clínico.
- Situação da pessoa na linha: a pior entre as boas práticas rastreadas (`untracked` não piora).
- Fontes: consulta finalizada (19a) com o profissional da linha; pressão e peso/altura registrados na escuta (18) ou na consulta; exame pedido ou com resultado (21a/21b) por SIGTAP; procedimento realizado (22) por SIGTAP.
- Desempenho: lista paginada por equipe, avaliação em lote por fonte (uma consulta por fonte por página).

## 6. Telas

- **Equipe (dashboard):** sugestões a confirmar (aceitar escolhendo a equipe; recusar com motivo); painel nominal por equipe/unidade e linha com a situação de cada pessoa e de cada boa prática; filtro "busca ativa" (alguma vencida); registrar contato; atalho para agendar (17); inscrever manualmente; encerrar com motivo. Painel não faz polling (lê com trilha).
- **Gestão:** prévia de C2–C7 por equipe (numerador/denominador estimados pelas boas práticas rastreadas, com aviso "estimativa — o valor oficial é calculado pelo SIAPS; visitas e vacinas não entram").
- **Admin:** adotar linhas do catálogo-base, criar linha própria.
- **Cidadão (wpda):** declarar condição de saúde no perfil (opcional, retirável); "Meu acompanhamento" (só par validado): linhas em que está inscrito e o que está para fazer, em linguagem simples ("está na hora de medir a pressão"), sem mostrar código nem diagnóstico que não seja o nome da linha; aviso de triagem periódica vencida (oferta do módulo 15 com intervalo de repetição passado).
- **Maintenance:** interruptor; catálogo-base (versões).

## 7. Segurança e LGPD

- Leitura nominal com trilha (`clinical_record.viewed` com `access: care_line`); nada de polling.
- Condição declarada e nota do contato cifradas; eventos só com ids: `care_line_enrollment.created`, `.ended`, `care_line_suggestion.accepted`, `.dismissed`, `care_outreach.recorded`, `citizen_condition.declared`, `.withdrawn`.
- Cidadão nunca vê sugestão nem motivo de encerramento; vê só as linhas em que foi inscrito.
- Inscrição e contatos contam como documento clínico na exclusão LGPD (regra da api#84); a condição declarada é apagada e não retém.

## 8. Testes

- **api:** elegibilidade e sugestão por fonte; confirmação obrigatória; avaliação de cada boa prática das seis linhas com séries conhecidas (janela, contagem mínima, âncora); `untracked` não piora; prévia por equipe; trilha; exclusão LGPD; guarda R18.
- **dashboard e wpda:** Vitest.
- **Prova no navegador:** sugestão por problema ativo e confirmação; diabético com consulta em dia e HbA1c vencida aparecendo na busca ativa; registro de contato e agendamento; gestante com âncora pela DUM; cidadão declarando condição e vendo "Meu acompanhamento"; triagem periódica vencida; prévia por equipe.

## 9. Entrega

0. Prova técnica: conferir as notas metodológicas C2–C7 vigentes (2025/NT 8/2026) e fixar as boas práticas, janelas e códigos de elegibilidade de cada linha.
1. `api`. 2. `dashboard`. 3. `wpda`. 4. `maintenance`. 5. catálogo público.

## 10. F-IDs (propostos)

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-23.1 | Interruptor e catálogo-base das linhas | api, maintenance |
| F-23.2 | Linhas da cidade (adoção e linhas próprias) | api, dashboard |
| F-23.3 | Sugestão e inscrição confirmada | api, dashboard |
| F-23.4 | Acompanhamento das boas práticas | api |
| F-23.5 | Painel da equipe e busca ativa | api, dashboard |
| F-23.6 | Prévia dos indicadores C2–C7 | api, dashboard |
| F-23.7 | Condição declarada e "Meu acompanhamento" no wpda | api, wpda |
| F-23.8 | Aviso de triagem periódica vencida | api, wpda |

## 11. Riscos

- Notas metodológicas mudam (revisão anual): o catálogo-base é versionado.
- Prévia diverge do oficial (visitas, vacinas, dados do PEC de outras unidades): rótulo de estimativa sempre visível.
- Desempenho do painel com equipes grandes (avaliação em lote, paginação).
