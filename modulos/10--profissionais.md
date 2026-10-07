# Módulo 10 — Profissionais

- **Estado:** Fechado
- **Tipo:** MVP

## Escopo

Profissionais de saúde da cidade como dado da plataforma: perfil ligado ao
usuário, vínculo com as unidades com a ocupação (CBO) de cada lugar, turnos com
data, e a chamada e o desfecho clínico restritos a quem tem vínculo com a
unidade do atendimento (ADR 0021).

**Entregue (F-10.1 a F-10.5):**
- perfil 1:1 com o usuário que tem o papel `health_professional`: nome
  profissional, conselho, UF e registro, CNS (validado e cifrado), telefone e
  e-mail de contato;
- o próprio profissional vê o cadastro e edita só nome e contato;
- vínculo com unidade e CBO, com início e fim, só por acréscimo; abrir e
  encerrar pedem step-up; o CBO exige o conselho do perfil quando declara um;
- turnos com instantes de início e fim (até 24h, sem sobreposição por
  profissional); encerrar vínculo cancela os turnos futuros;
- chamar e registrar desfecho clínico exigem papel e vínculo ativo com a unidade
  (`missing_role` / `missing_link`); o turno nunca bloqueia; a fila mostra o
  nome profissional de quem chamou;
- telas: Profissionais (lista, pendências, ficha), Meu perfil, etiqueta de
  pendência na Equipe e Atendimento com ações clínicas só na unidade vinculada;
- semente de dev com profissionais fictícios de forma real.

**Fora, por enquanto:** CNES (F-10.6), escalonamento do alerta urgente
(F-10.7), ocupações sem conselho (agente comunitário, agente de endemias),
semana-padrão gerando turnos e a agenda de vagas consumindo os turnos
(módulo 08).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0021 | Perfil 1:1 com o usuário, vínculo com unidade e CBO com início e fim, turnos com instantes, chamada restrita ao vínculo |
| 0019 | Papel `health_professional`: chama e registra o desfecho |

Ainda a decidir: integração com o CNES e o papel do profissional no
escalonamento do alerta urgente (em aberto no ADR 0021).

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `professionals`, `professional_links`, `professional_shifts`; lista CBO versionada; rotas sob `/professionals`; regra da chamada em `Attendances::Call`/`CallNext`/`Close` |
| `admin` | — |
| `dashboard` | Profissionais (lista, pendências, ficha com perfil, vínculos e turnos), Meu perfil, etiqueta na Equipe, Atendimento por vínculo |
| `wpda` | — |

## Pré-requisitos

- Módulo 09 (Unidades) — profissional vincula a unidade(s).
- Módulo 06 (Identidade/Acesso) — o profissional é um usuário com o papel
  `health_professional` e um perfil 1:1 (ADR 0021).

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-10.1 | Cadastro do profissional da cidade (nome, conselho e registro, CNS) ligado ao usuário com papel `health_professional` | api, dashboard | 0021 |
| F-10.2 | Vínculo do profissional com uma ou mais unidades | api, dashboard | 0021 |
| F-10.3 | Ocupação do profissional por vínculo (código CBO da saúde) | api, dashboard | 0021 |
| F-10.4 | Turnos com data por vínculo, base da agenda de vagas do módulo 08 | api, dashboard | 0021 |
| F-10.5 | Chamada e desfecho só por profissional vinculado à unidade do atendimento, com o profissional gravado no atendimento | api, dashboard | 0021, 0019 |
| F-10.6 | Importação e consulta do cadastro nacional de profissionais (CNES) (substituída pela F-16.5, Ciclo 2) | api, dashboard | 0028 |
| F-10.7 | Profissional como destino do escalonamento de alerta urgente (fora do Ciclo 1) | api | 0006 |

F-10.1 a F-10.5 entregues (ADR 0021).

**Fora do Ciclo 1 (decisão do usuário, 2026-09-28):** F-10.6 (CNES) e F-10.7
(escalonamento do alerta urgente). Os dois dependem de pontos em aberto: a
fonte e o escopo do CNES (API do DATASUS ou arquivo; importar ou só consultar)
e o conceito de plantão do ADR 0006. Continuam no registro de F-IDs e no board
como `Not Started`, marcados "fora do Ciclo 1", e não contam para o
fechamento do módulo neste ciclo.

## Riscos herdados

- **Rollout:** o api com a regra da chamada, publicado numa cidade sem perfis e
  vínculos cadastrados, deixa todo profissional sem chamar. Seguir
  [`operacao/rollout-profissionais.md`](../operacao/rollout-profissionais.md)
  (api `8ee7da4` → dashboard → cadastro → api final).
- **Infra:** a sobreposição de turnos é garantida por EXCLUDE com `btree_gist`;
  o banco de produção precisa de `postgresql-contrib`.
- **Operacional:** turno lançado um a um dá trabalho à unidade (custo aceito no
  ADR 0021).
- **Operacional:** desativar unidade não encerra os vínculos dela; sem efeito
  prático, porque unidade inativa não recebe check-in. A etiqueta da Equipe
  mostra "ok" para quem só tem vínculo em unidade desativada.
- **Em aberto:** CNES, escalonamento do alerta, ocupações sem conselho.

## Critério de fechamento do módulo

- F-10.1 a F-10.5 verificadas. F-10.6 foi substituída pela F-16.5 (Ciclo 2)
  e está `Verified` por isso; F-10.7 está fora do Ciclo 1. Nenhuma das duas
  conta para este fechamento.
- Suíte de invariante (`spec/invariants/professional_invariants_spec.rb`, com
  teste de mutação): perfil 1:1 com o usuário; vínculo e turno só por acréscimo;
  um vínculo ativo por (profissional, unidade, CBO); turnos sem sobreposição e
  até 24h; sem turno em vínculo encerrado; só o `municipal_admin` cadastra;
  chamada e desfecho exigem papel e vínculo com a unidade; turno nunca bloqueia
  ato clínico; eventos sem dado sensível.

## Histórico

- 2026-09-27 — F-10.1 a F-10.5 entregues. api mergeado em `66ab5fc` (suíte
  2391/0; invariantes com 11 mutações, todas vermelhas); dashboard mergeado em
  `f750c41` (416 testes). A prova no navegador (admin e profissional) achou um
  defeito anterior ao módulo: quem tinha só o papel `health_professional`
  recebia 403 na lista de unidades e nunca chegava à fila; corrigido no api em
  `a2d20c8`. Spec e planos em `superpowers/specs/2026-09-27-module-10-professionals-design.md`
  e `superpowers/plans/2026-09-27-module-10-professionals-*.md`. Módulo passa de
  `Planejado` a `Em andamento`.
- 2026-09-28 — F-10.6 (CNES) e F-10.7 (escalonamento do alerta) tirados do
  Ciclo 1 por decisão do usuário; o critério de fechamento passa a cobrir
  F-10.1 a F-10.5.
- 2026-09-28 — Verificação do módulo (dossiê por F-ID,
  [`relatorios/2026-09-28-verificacao-modulo-10.md`](../relatorios/2026-09-28-verificacao-modulo-10.md)):
  F-10.1 a F-10.5 `Verified` por aprovação do usuário, sem lacuna bloqueante
  (api 218 exemplos e dashboard 132 testes rodados na verificação, 0 falhas).
  Critério de fechamento do Ciclo 1 cumprido; módulo `Fechado`. F-10.6 e
  F-10.7 seguem `Not Started`, fora do Ciclo 1.
- 2026-10-07 — F-10.6 (CNES) fechada como **substituída pela F-16.5**
  (módulo 16, ADR 0028, Ciclo 2), por decisão do usuário: a importação do
  CNES, com equipes e casamento confirmado pela cidade, foi entregue e
  verificada lá. O card da F-10.6 foi para `Verified` com essa nota, e a
  pendência docs#4 foi fechada. F-10.7 segue fora do Ciclo 1.
