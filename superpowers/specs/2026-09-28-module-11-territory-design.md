# Módulo 11 — Território (F-11.1 a F-11.7) — design

**Data:** 2026-09-28
**Status:** aprovado em conversa (2026-09-28), aguardando revisão do texto
**Afeta:**
- `apps/api`:
  - banco de cada cidade: `neighborhoods` e `neighborhood_coverages` novas; colunas novas em `health_units`, `citizens` e `triages`; trigger de imutabilidade do bairro da triagem;
  - semente por cidade (`db/seeds/territory/<slug>.yml`) e rake `city:territory:seed`;
  - rotas sob o prefixo único `/territory` (escrita, só `municipal_admin`);
  - `/citizen`: lista de bairros, bairro da pessoa, unidade de referência;
  - `/attendance/units`: endereço; leitura do atendimento com `reference_unit_ids`;
  - 5 queries de `/admin/api` com filtro opcional por bairro e supressão.
- `apps/dashboard`: módulo "Território" (admin), endereço com CEP no formulário de unidade, pré-seleção no desfecho, seletor de bairro em 5 painéis.
- `apps/wpda`: bairro na escolha da pessoa, "Trocar bairro", bloco "Sua unidade de referência".
- `apps/admin`: nada.

**ADR:** `docs/adr/0023.md` · **Módulo:** `docs/modulos/11--territorio.md`

**Fora desta entrega:** região acima do bairro, geometria (PostGIS), distância, importação de bairros por CSV, restrição de agendamento à unidade de referência, horário de funcionamento da unidade.

## 1. Ponto de partida

- **Nada territorial existe.** `health_units` tem só `name`, `kind`, `active`. `citizens` tem `cpf`, `phone`, `verification_level`. `city_profile` tem `ibge_code` e `uf`.
- **Módulo 09 / ADR 0018** deixaram "endereço e geolocalização" para o módulo 11, "sem refazer a tabela".
- **Triagem:** nasce em `StartTriage.call(conversation:)`, usado pela web (`Citizens::StartConversation`) e pelo WhatsApp (descontinuado). `conversation.citizen` pode ser nulo em conversas antigas do WhatsApp.
- **Cidadão web:** `POST /citizen/conversations` resolve ou cria o `Citizen` (`RegisterPerson`) **depois** de conferir o consentimento vigente; `GET /citizen/people` lista os CPFs da sessão.
- **Pedido de agendamento:** nasce no desfecho do atendimento, no dashboard. `Attendances::Close` recebe `referral_unit_id` e chama `AppointmentRequests::Lifecycle.open_for!`. No "retorno" o destino é sempre a própria unidade. No wpda o cidadão só confirma ou cancela.
- **Chaves que já existem:** `appointment_requests.root_triage_id` (NOT NULL), `attendances.triage_id` (opcional) e `attendances.appointment_id`, `report_snapshots.triage_id`, `triages.conversation_id`, `conversations.citizen_id`.
- **Painéis:** `/admin/api/*` é só leitura e é consumido pelo dashboard **e** pelo console `admin` (ADR 0022). Os painéis com cidadão são Visão geral, Classificação, Triagens, Relatórios e Conversas. Filas (`QueuesQuery`) é a fila de jobs do Solid Queue e não tem cidadão.
- **PostGIS** não é extensão *trusted*; o papel dono do banco da cidade não a cria.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| D1 | Uso no Ciclo 1: indicar a unidade de referência do cidadão e recortar os painéis por bairro. Sem mapa, sem PostGIS. |
| D2 | Bairros: semente própria por cidade como carga inicial; depois o `municipal_admin` cria, renomeia, desativa, reativa e edita a cobertura no dashboard. A carga nunca sobrescreve o que foi editado. |
| D3 | O cidadão informa o bairro uma vez, na escolha da pessoa no wpda, e pode trocar. "Prefiro não informar" é permitido. A triagem copia o bairro na criação. |
| D4 | Unidade de referência: mostrada no resultado da triagem, só na área logada do cidadão (nunca no relatório público, decisão de 2026-09-28); pré-selecionada como destino no desfecho "encaminhado". Várias unidades: todas; nenhuma: o bloco some. Nunca restringe. |
| D5 | Filtro de bairro nos painéis com cidadão (5, ver §1), parâmetro opcional, com supressão de 1 a 4 quando o filtro está ligado. |
| D6 | Endereço da unidade em texto; CEP consultado pelo navegador do dashboard direto no ViaCEP; falha = preenchimento à mão; o bairro do CEP é só sugestão. |
| D7 | Abordagem 1: uma cópia só, na triagem; os outros painéis chegam ao bairro pelas chaves existentes; conversa e atendimento sem triagem usam o bairro atual do cidadão. |
| D8 | Só o `municipal_admin` edita território e endereço, sem step-up. |

## 3. Dados

Uma migração em `db/city_migrate`, só de expansão e reversível.

### 3.1 `neighborhoods`

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | uuid | |
| `name` | string | NOT NULL, sem espaços nas pontas, não vazio, até 120; índice único em `lower(name)` |
| `active` | boolean | NOT NULL, default true |
| `source` | string | NOT NULL, `seed` ou `manual` (check) |
| `seed_key` | string | opcional; único quando presente; chave estável do item da semente (decisão de 2026-09-28) |
| timestamps | | |

### 3.2 `neighborhood_coverages`

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | uuid | |
| `neighborhood_id` | uuid | NOT NULL, FK |
| `health_unit_id` | uuid | NOT NULL, FK |
| `created_at` | datetime | |

Índice único em (`neighborhood_id`, `health_unit_id`).

### 3.3 Colunas novas

| Tabela | Coluna | Regra |
|---|---|---|
| `health_units` | `address_street` (string, até 160), `address_number` (string, até 20), `address_complement` (string, até 80) | opcionais |
| `health_units` | `address_zip` | opcional; exatamente 8 dígitos (check) |
| `health_units` | `neighborhood_id` | opcional, FK; onde a unidade fica |
| `citizens` | `neighborhood_id` | opcional, FK, índice |
| `triages` | `neighborhood_id` | opcional, FK, índice (`neighborhood_id`, `created_at`) |

**Trigger de cidade** (em `db/city_triggers.sql`, no padrão do snapshot do módulo 04): `UPDATE` em `triages` que mude `neighborhood_id` levanta erro. A cópia acontece só no `INSERT`. Única exceção (decisão de 2026-09-28): mudar para `NULL` quando a triagem é anonimizada por revogação de consentimento; a anonimização zera o bairro.

### 3.4 Semente por cidade

`db/seeds/territory/<slug>.yml`:

```yaml
neighborhoods:
  - name: Santa Felicidade
    units: ["UBS Santa Felicidade"]
  - name: Batel
    units: []
```

Rake `city:territory:seed[slug]` (e `city:territory:seed:all`), dentro do contexto da cidade:
- cada item do YAML tem `key` estável; a carga casa por `seed_key`, nunca por nome, então renomear no dashboard não gera duplicado;
- chave ausente no banco → cria com `source: seed` e `seed_key`; se já houver bairro manual com o mesmo nome (`lower`), não cria: aviso;
- bairro existente → não toca (nem nome, nem ativo);
- cobertura: cria o par só quando o bairro foi **criado nesta carga** e a unidade existe e está ativa pelo nome; unidade não encontrada vira aviso na saída, não erro;
- arquivo ausente → mensagem e saída sem erro;
- relatório no fim: criados, já existentes, avisos.

Sementes: Curitiba e Maringá com bairros reais (lista oficial das prefeituras) e cobertura ligando as unidades da semente de dev (ver [semente de dev realista](#8-semente-de-dev)).

### 3.5 Eventos

`neighborhood.created`, `neighborhood.renamed`, `neighborhood.deactivated`, `neighborhood.activated`, `neighborhood.coverage_changed` (ids adicionados e removidos), `citizen.neighborhood_changed` (`citizen_id`, `from_id`, `to_id`). Só ids no payload. Se alguma guarda do `api` exigir declarar nomes de evento, declarar os novos (nunca afrouxar a guarda).

## 4. API

### 4.1 Rotas

**Território** (`/territory`, papel `municipal_admin`; outro papel = 403 `missing_role`; uma entrada nova no proxy do Vite do dashboard):

| Método e rota | Faz |
|---|---|
| `GET /territory/neighborhoods` | todos, inclusive inativos, com `source`, `active` e `units: [{id, name, active}]` |
| `POST /territory/neighborhoods` | `{ name }` → cria com `source: manual` |
| `POST /territory/neighborhoods/:id` | `{ name }` → renomeia |
| `POST /territory/neighborhoods/:id/deactivate` · `/activate` | muda `active` |
| `POST /territory/neighborhoods/:id/coverage` | `{ health_unit_ids: [...] }` → substitui o conjunto |

Recusas: `name_taken` (422), `blank_name` (422), `inactive_unit` (422, unidade inativa ou inexistente na cobertura), `inactive_neighborhood` (422, cobertura em bairro inativo), `not_found` (404).

**Cidadão** (`/citizen`, sessão por cookie):

| Método e rota | Faz |
|---|---|
| `GET /citizen/neighborhoods` | bairros ativos, por nome: `{ neighborhoods: [{id, name}] }` |
| `GET /citizen/people` | cada pessoa ganha `neighborhood: {id, name} \| null` |
| `POST /citizen/conversations` | aceita `neighborhood_id` opcional (regra em §4.2) |
| `POST /citizen/people/:id/neighborhood` | `{ neighborhood_id \| null }`; CPF fora da sessão = 404; bairro inativo ou inexistente = 422 `invalid_neighborhood` |
| `GET /citizen/triages/:id` | ganha `reference_units: [{id, name, kind, address: {street, number, complement, zip}}]` (campos de `address` podem ser nulos). O relatório público (link sem login) **não** ganha, para não revelar o bairro |

**Unidades** (`/attendance/units`): `create` e `update` aceitam `address_street`, `address_number`, `address_complement`, `address_zip`, `neighborhood_id`. CEP fora de 8 dígitos = 422 `invalid_zip`; bairro inexistente = 422 `invalid_neighborhood`. As leituras devolvem os campos.

**Atendimento:** cada linha de `GET /attendance/units/:id/queue` (que alimenta o formulário de desfecho) ganha `reference_unit_ids` (unidades ativas que cobrem o bairro **da triagem** do atendimento; sem triagem, o bairro atual do cidadão), **sem a própria unidade do atendimento** (decisão de 2026-09-28: encaminhar para si mesma não faz sentido; para isso há o "retorno").

**Lista para o filtro:** `GET /admin/api/neighborhoods` → `{ neighborhoods: [{id, name, active}] }`, só leitura, mesma autorização dos demais `/admin/api` da cidade: o filtro vale para todos os papéis que leem os painéis (decisão de 2026-09-28).

**Painéis** (`/admin/api/overview`, `classification`, `triages`, `reports`, `conversations`): aceitam `neighborhood_id=<uuid>` ou `neighborhood_id=none`. Parâmetro inválido = 422 `invalid_neighborhood`. A resposta ganha `filter: { neighborhood: {id, name} | "none" | null }`.

### 4.2 Comandos e regras

- `Territory::CreateNeighborhood`, `RenameNeighborhood`, `SetNeighborhoodActive`, `ReplaceCoverage` em `app/commands/territory/`, cada um numa transação com o evento.
- `ReplaceCoverage` trava o bairro (`FOR UPDATE`), confere que todas as unidades estão ativas e calcula adicionados e removidos.
- `Citizens::SetNeighborhood(citizen:, neighborhood_id:)` — aceita `nil`; recusa bairro inativo; publica o evento só se mudou.
- `POST /citizen/conversations`: depois de conferir o consentimento e resolver o cidadão, se `neighborhood_id` veio e o cidadão **não tem** bairro, grava via `Citizens::SetNeighborhood`. Se já tem, ignora (a troca é pela rota própria). Bairro inválido = 422 `invalid_neighborhood`, **antes** de criar o cidadão.
- `StartTriage` copia `conversation.citizen&.neighborhood_id` no `create!`.
- `Territory::ReferenceUnits.for(neighborhood_id)` — única fonte da unidade de referência: unidades ativas com cobertura no bairro, por nome. Bairro nulo = lista vazia. Usada pelo cidadão e pelo atendimento.

### 4.3 Filtro e supressão nos painéis

`Admin::NeighborhoodFilter` (objeto único, usado pelas 5 queries):

| Painel | Base | Como chega ao bairro |
|---|---|---|
| Visão geral | triagens (concluídas, urgentes, conclusão); conversas ativas | `triages.neighborhood_id`; conversa → `citizens.neighborhood_id` |
| Classificação | triagens | `triages.neighborhood_id` |
| Triagens | triagens | `triages.neighborhood_id` |
| Relatórios | `report_snapshots` | join `triages` |
| Conversas | conversas; tempo médio de triagem | conversa → cidadão; triagem → `triages.neighborhood_id` |

`none` = coluna nula (inclui conversas sem cidadão). KPIs que não são do cidadão (jobs com falha na Visão geral) ignoram o filtro e não são suprimidos.

**Supressão** (só com filtro ligado), aplicada **depois** de agregar, por `Admin::SmallCount.wrap(n)`:
- 1 a 4 → `{ suppressed: true }`; 0 e ≥ 5 → o número;
- vale para KPI, contagem por categoria e pontos de série (sparkline);
- percentuais e médias calculados sobre um total de 1 a 4 → `{ suppressed: true }`;
- listas de amostra (`sampleTriages`, linhas de relatório) vêm como `null` quando o total filtrado do painel estiver suprimido;
- KPI com valor suprimido vem com `delta: null`.

O console `admin` não envia o parâmetro, então nunca vê `{ suppressed: true }`.

## 5. Dashboard

- **Território** (menu, só `municipal_admin`): tabela de bairros (nome, origem, estado, nº de unidades), busca por nome, criar e renomear em diálogo, desativar e reativar, e a cobertura em caixas de seleção das unidades ativas. Erros da API com texto em português.
- **Formulário de unidade** (módulo 09): campos de endereço; ao completar 8 dígitos de CEP, o navegador chama `https://viacep.com.br/ws/<cep>/json/` com timeout de 5 s; sucesso preenche logradouro e mostra "bairro segundo o CEP: X" como sugestão; o admin escolhe o bairro na lista da cidade (pré-seleciona se o nome bate, sem diferenciar maiúsculas e acentos). Erro, `{erro: true}` ou timeout: aviso "não foi possível consultar o CEP" e campos livres. Se houver CSP no deploy do dashboard, liberar `connect-src https://viacep.com.br`.
- **Desfecho "encaminhado":** a unidade de destino já vem com a primeira de `reference_unit_ids` por nome (nunca a própria unidade; se só ela for de referência, nada vem escolhido); as outras de referência sobem para o topo da lista com a etiqueta "referência".
- **Seletor de bairro** nos 5 painéis, para todos os papéis que os leem (lista de `GET /admin/api/neighborhoods`): "Todos", "Sem bairro" e os bairros (inativos marcados), guardado na URL (`?bairro=`); valor suprimido aparece como "< 5" com uma dica explicando a regra.
- O cache já é por usuário (`useSessionQueryClient`); as chaves de consulta levam o bairro.

## 6. wpda

- **"Para quem é esta triagem?":** CPF novo → escolhe o bairro numa lista com busca, junto com o CPF, com "Prefiro não informar". Pessoa existente sem bairro → a pergunta aparece uma vez antes de começar. O `neighborhood_id` vai no `POST /citizen/conversations`.
- **"Trocar bairro"** ao lado de cada pessoa, usando `POST /citizen/people/:id/neighborhood`.
- **Resultado da triagem (área logada):** bloco "Sua unidade de referência" com nome, tipo e endereço de cada unidade de `reference_units` de `GET /citizen/triages/:id`; lista vazia = bloco ausente. O relatório público (`/r/:token`) não mostra o bloco.

## 7. Testes

### 7.1 Invariantes: `spec/invariants/territory_invariants_spec.rb`

- `UPDATE` do bairro de uma triagem levanta erro no banco, exceto para `NULL` na anonimização por revogação.
- Bairro inativo não entra em cobertura nem no cidadão.
- `Territory::ReferenceUnits` nunca devolve unidade inativa.
- O relatório público nunca contém `reference_units` nem bairro.
- Com filtro, nenhum dos 5 painéis devolve um número de 1 a 4 em lugar algum do JSON (varredura recursiva).
- Rodar a semente duas vezes não muda nada; rodar depois de editar não desfaz a edição.
- Nenhum código do `api` referencia `viacep` (varredura do fonte).

### 7.2 Demais

- **api (RSpec, TDD):** modelos e comandos de território; request specs de `/territory` (papel, recusas), `/citizen` (bairro inválido antes de criar CPF, sem gravação sem consentimento, 404 para CPF alheio, troca não altera triagem antiga), unidades com endereço, `reference_unit_ids` no atendimento, os 5 painéis com e sem filtro e com `none`; spec do rake. Specs de request com `type: :request`.
- **dashboard (vitest):** tela Território; formulário com ViaCEP simulado (sucesso, `{erro: true}`, falha de rede, timeout); pré-seleção no desfecho; seletor e renderização de "< 5".
- **wpda (vitest):** escolha do bairro com CPF novo e pessoa existente; "Prefiro não informar"; troca de bairro; bloco de referência com 0, 1 e 2 unidades.
- **Prova no navegador** a partir dos worktrees (api em :3031 e Vite do worktree): admin cadastra bairro e cobertura e endereço com CEP → cidadão escolhe bairro e faz triagem → vê a unidade de referência → profissional encerra com "encaminhado" e o destino vem pré-escolhido → painel filtrado mostra "< 5". O login e o step-up são feitos pelo usuário.
- **Custo:** conferir o tempo por exemplo da suíte do api (~0,10 s com o host calmo) antes do merge.

## 8. Semente de dev

Dados fictícios de forma real: bairros reais de Curitiba e Maringá; cada unidade da semente de dev ganha endereço com CEP real do bairro e cobre de 2 a 4 bairros; alguns bairros ficam sem cobertura; os cidadãos de demonstração ganham bairros variados, e alguns ficam sem bairro, para o painel mostrar "Sem bairro" e "< 5".

## 9. Fatias e entrega

| Fatia | Conteúdo | F-IDs |
|---|---|---|
| 1 | api: migração, trigger, modelos, comandos, `/territory`, semente e rake | F-11.1, F-11.2 |
| 2 | api + dashboard: endereço da unidade; tela Território; CEP | F-11.3 (e telas de F-11.1/.2) |
| 3 | api + wpda: bairro do cidadão, cópia na triagem, unidade de referência | F-11.4, F-11.5 |
| 4 | api + dashboard: `reference_unit_ids` e pré-seleção no desfecho | F-11.6 |
| 5 | api + dashboard: filtro e supressão nos 5 painéis | F-11.7 |

Planos: um por app (`api`, `dashboard`, `wpda`), em `docs/superpowers/plans/2026-09-28-module-11-territory-*.md`. Trabalho em worktree a partir de `origin/main` em cada repo. Ordem de merge: api antes de dashboard e wpda (os fronts leem campos novos).

**Rollout** (runbook `operacao/rollout-territorio.md`): migração só de expansão, sem ordem rígida de imagens. (1) imagem do api + `city:migrate:all`; (2) `city:territory:seed[slug]` nas cidades com semente; (3) dashboard e wpda. Sem semente, nada quebra: o cidadão fica sem bairro e o bloco de referência não aparece.

**Docs:** ADR 0023; `modulos/11--territorio.md` (`Stub` → `Planejado`, F-IDs, superfícies, critério de fechamento); `modulos/09--unidades.md` (endereço entregue pelo 11); índice `modulos/README.md`; CSV `funcionalidades-mvp.csv`; `adr/README.md`; guarda `adr_pointers_spec` do api com `VALID_RANGE` 1..23; 7 cards no board (mod-11).

## 10. Riscos

- **Qualidade da semente:** lista de bairros incompleta ou com grafia diferente da prefeitura; mitigado pela edição no dashboard.
- **ViaCEP indisponível ou alterado:** o preenchimento à mão é sempre possível.
- **Supressão não impede todo cruzamento:** filtrar por bairro e alternar períodos pode, em tese, isolar casos por diferença. Aceito no Ciclo 1; o painel é restrito a papéis da própria cidade.
- **Cidadão que não informa bairro** fica fora da indicação e cai em "Sem bairro"; a taxa de "Sem bairro" mostra se a pergunta está funcionando.
- **Contrato de `/admin/api`:** o parâmetro é opcional, mas as 5 queries mudam; o console `admin` precisa continuar passando no teste de contrato.
