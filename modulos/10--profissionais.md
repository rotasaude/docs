# Módulo 10 — Profissionais

- **Estado:** Planejado
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Cadastro de profissionais de saúde da cidade, vínculo com unidades,
especialidades, agenda de atendimento. Base para alocação de paciente
após triagem e para escalonamento de alerta urgente.

## ADRs governantes

A definir. Provavelmente:

- ADR de modelo de profissional (CNS, conselho de classe, registro).
- ADR de integração com cadastro nacional (CNES).

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | Modelo `professionals`, vínculo com `health_units` |
| `admin` | — |
| `dashboard` | CRUD de profissionais da cidade |
| `wpda` | Identificação do profissional na ficha do paciente |

## Pré-requisitos

- Módulo 09 (Unidades) — profissional vincula a unidade(s).
- Módulo 06 (Identidade/Acesso) — profissional pode ou não ser usuário do
  sistema; decisão a tomar.

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-10.1 | Cadastro do profissional da cidade (nome, conselho e registro, CNS) ligado ao usuário com papel `health_professional` | api, dashboard | — |
| F-10.2 | Vínculo do profissional com uma ou mais unidades | api, dashboard | 0019 |
| F-10.3 | Especialidades do profissional | api, dashboard | 0019 |
| F-10.4 | Escala de atendimento por unidade (dias e horários), base da agenda de vagas do módulo 08 | api, dashboard | 0019 |
| F-10.5 | Chamada e desfecho só por profissional vinculado à unidade do atendimento, com o profissional gravado no atendimento | api, dashboard | 0019 |
| F-10.6 | Importação e consulta do cadastro nacional de profissionais (CNES) | api, dashboard | — |
| F-10.7 | Profissional como destino do escalonamento de alerta urgente | api | 0006 |

Todas planejadas: nenhuma tem ADR próprio ainda. As marcadas 0019 vêm dos itens em aberto dele; a F-10.7 toca a priorização clínica do ADR 0006.

## Histórico
