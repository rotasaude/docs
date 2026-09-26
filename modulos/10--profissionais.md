# Módulo 10 — Profissionais

- **Estado:** Planejado
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Cadastro de profissionais de saúde da cidade, vínculo com unidades,
especialidades, agenda de atendimento. Base para alocação de paciente
após triagem e para escalonamento de alerta urgente.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0021 | Perfil 1:1 com o usuário, vínculo com unidade e CBO com início e fim, turnos com data, chamada restrita ao vínculo |
| 0019 | Papel `health_professional`: chama e registra o desfecho |

Ainda a decidir: integração com o CNES e o papel do profissional no escalonamento do alerta urgente (em aberto no ADR 0021).

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | Modelo `professionals`, vínculo com `health_units` |
| `admin` | — |
| `dashboard` | CRUD de profissionais da cidade |
| `wpda` | Identificação do profissional na ficha do paciente |

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

Todas planejadas. F-10.1 a F-10.5 seguem o ADR 0021; F-10.6 (CNES) e F-10.7 (escalonamento) ficaram em aberto nele.

## Histórico
