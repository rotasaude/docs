# Ciclo de desenvolvimento — Rota Saúde

Documento mestre. Não é ADR (não decide arquitetura, organiza o trabalho que a
consome). Não é prosa de leitura: é referência operacional.

## Hierarquia de artefato

Quatro níveis, cada um respondendo a uma pergunta distinta. Não se sobrepõem.

| Nível | Pergunta | Onde vive | Imutável? |
|---|---|---|---|
| **ADR** | Por que decidimos assim? | `docs/adr/` | Sim — ADR novo em vez de editar |
| **Módulo** | Que fatia coerente do sistema agrupa decisões? | `docs/modulos/` | Não |
| **Funcionalidade** | Que capacidade observável o módulo entrega? | issue no board | Não |
| **Tarefa / PR** | Que mudança discreta materializa a funcionalidade? | issue-filha + PR | Não |

## Quatro aplicações (ADR 0002)

Cada aplicação é um repositório próprio, com deploy, time e permissões
independentes (ADR 0002).

| Repo | Papel | Tenant scope | Conexão |
|---|---|---|---|
| `api` | backend Rails (web + worker, mesma imagem) | varia por caminho | `rota_app` / `rota_admin` |
| `admin` | plataforma, dono `platform_operator` | cross-tenant | `rota_admin` (BYPASSRLS) |
| `dashboard` | cidade, dono `municipal_admin` / `viewer` | tenant-scoped | `rota_app` (RLS) |
| `wpda` | cidadão, jornada do paciente | tenant-scoped | `rota_app` (RLS) |

## 14 módulos

**MVP (7):** WhatsApp · Conversação · Triagem · Relatórios · Dashboard ·
Identidade/Acesso · LGPD/Auditoria.

**Estratégicos (7):** Agendamento · Unidades · Profissionais · Território ·
Campanhas · Acompanhamento · Analytics.

Índice e mapa em `docs/modulos/README.md`.

## Ciclo por funcionalidade (5 passos)

Cada funcionalidade percorre os cinco passos em ordem. Gates duros entre eles.

### 1. Escopo

Issue de Funcionalidade aberta com:

- módulo (de `docs/modulos/`)
- superfície(s) afetada(s) (`api`, `admin`, `dashboard`, `wpda`)
- ADRs governantes (referência para o `docs/adr/`)
- critério de aceite enumerado
- camadas de teste exigidas (ver passo 4)
- out-of-scope explícito

Sem qualquer desses campos, a issue não sai de `Não iniciado`.

### 2. Desenho

Se a funcionalidade introduz decisão arquitetural nova: **pausa** e abre ADR.
Não passa para implementação até o ADR estar aceito (ou Proposta com escopo
fechado).

Se consome decisão existente: segue direto para o passo 3, anotando os ADRs
consumidos na issue.

### 3. Implementação

- Código no repo correto (tabela acima).
- Branch nomeada `mod-<n>/<slug-funcionalidade>` (ex.: `mod-3/scoring-weighted`).
- PR amarrado à issue com `Closes #N` (board cross-repo da organização).
- PR descreve quais ADRs consome (por URL — ADR 0002); quais invariantes
  preserva.

### 4. Teste

Três camadas mínimas, todas presentes:

| Camada | O que cobre | Exemplo |
|---|---|---|
| **Invariante** | O que o ADR exige nunca violar | RLS isola tenant; write+enqueue atômico |
| **Comportamental** | Caso feliz + bordas | Conversa avança; resposta inválida re-pergunta |
| **Regressão** | Bug corrigido (só se aplicável) | `Conversation.for` sem `municipality_id` levanta |

PR sem as camadas exigidas no escopo **não merge**.

### 5. Documentação

- README do módulo atualizado (`docs/modulos/<n>--<nome>.md`).
- Changelog da funcionalidade: 1-3 linhas em `## Histórico` do módulo.
- Se mudou contrato observável (API pública, schema, evento de domínio): nota
  no ADR-pai (campo "Funcionalidades que materializam"), nunca reescrita do ADR.

## Cinco gerências contínuas

Atravessam todas as funcionalidades. Não são etapas; são responsabilidades
sempre ativas.

| Gerência | Onde vive | Quem mexe | Cadência |
|---|---|---|---|
| **Entregas** | board: `Em curso` → `Implementado` → `Verificado` | autor / revisor | por funcionalidade |
| **Dívida** | board: cards `Divergente`, label `risco:*` | agente do drift report | semanal |
| **Funcionalidade** | issue tipo `feature`, vínculo ao módulo | autor + revisor | por funcionalidade |
| **Documentação** | `docs/modulos/`, README de cada um | autor | por funcionalidade |
| **Releases** | tag git + changelog do módulo | mantenedor | por marco de módulo |

### Releases não casam com calendário

Casam com **marco de módulo**. Exemplo de cadência possível do módulo Triagem:

- `triagem-v0.1` — fluxo + classificação `weighted`
- `triagem-v0.2` — `decision_table` + autoria
- `triagem-v0.3` — preview no dashboard

Cada release agrega as funcionalidades verificadas daquele módulo no período.
Não há release "do sistema inteiro" no MVP; cada módulo evolui sob versão
própria. Quando uma funcionalidade atravessa repos, cada repo libera no seu
ritmo (release coordenado cross-repo é item em aberto — ADR 0002).

## Gates resumidos

| Transição | Gate |
|---|---|
| `Não iniciado` → `Em curso` | Issue tem escopo completo (passo 1) |
| `Em curso` → `Implementado` | PR merge'do; 3 camadas de teste passando |
| `Implementado` → `Verificado` | Revisão humana; ADR-pai e doc do módulo atualizados |
| `Verificado` → release | Marco do módulo agrupa as funcionalidades verificadas |

## O que NÃO entra neste ciclo

- **Hotfix de produção** — caminho próprio (branch `hotfix/<n>`, PR direto,
  ADR retroativo se virou regra). Não passa por escopo formal.
- **Refactor sem mudança de comportamento** — issue tipo `chore`, sem ADR e
  sem camada de invariante nova (mantém as existentes passando).
- **Discussão de arquitetura nova** — vira ADR no repo `docs`, não issue no board.

## Disciplina mínima

- ADR aceita não se reescreve. Mudança = ADR novo.
- Funcionalidade não atravessa app sem ser ratificada pelo módulo.
- `Implementado` ≠ `Verificado`. Sem essa separação, dívida de verificação
  se esconde.
- Teste sem camada de invariante é teste incompleto.
