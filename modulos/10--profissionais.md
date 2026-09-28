# Módulo 10 — Profissionais

- **Estado:** Em andamento
- **Tipo:** Estratégico (pós-MVP)

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
| F-10.6 | Importação e consulta do cadastro nacional de profissionais (CNES) | api, dashboard | — |
| F-10.7 | Profissional como destino do escalonamento de alerta urgente | api | 0006 |

F-10.1 a F-10.5 entregues (ADR 0021). F-10.6 (CNES) e F-10.7 (escalonamento)
seguem em aberto no ADR.

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

- F-10.1 a F-10.7 verificadas (F-10.6 e F-10.7 dependem de decisão).
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
  `Planejado` a `Em andamento` (F-10.6 e F-10.7 seguem sem implementação).
