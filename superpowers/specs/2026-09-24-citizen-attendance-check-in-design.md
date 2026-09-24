# Check-in na unidade e desfecho do atendimento (canal web, subprojeto 3) — design

**Data:** 2026-09-24
**Status:** aprovado em conversa (2026-09-24), aguardando revisão do texto
**Afeta:** `apps/api` (banco de cada cidade: `health_units`, `attendances`, colunas novas em `citizen_verification_codes`; rotas `/attendance/*` e `/citizen/triages/:id/check_in_code`), `apps/dashboard` (módulo "Atendimento" ampliado; cadastro de unidades) e `apps/wpda` ("Cheguei na unidade" e situação do atendimento).
**Card:** [rotasaude/api#8](https://github.com/rotasaude/api/issues/8) · **ADR:** `docs/adr/0018.md`

**Origem:** o ADR 0017 previu que o cidadão confirma atendimento depois da triagem. O fluxo pedido: a triagem termina com "procure atendimento"; o cidadão chega a uma unidade de saúde e confirma a identidade; esse **check-in** fecha o ciclo da triagem e **inicia o atendimento** naquela unidade. O atendimento termina com um desfecho registrado pela unidade, que pode ser um encaminhamento.

## 1. Ponto de partida

- A triagem web já termina `completed`, com resultado congelado e relatório (subprojeto 1). O contrato de dados das triagens não muda.
- O subprojeto 2 criou o código do balcão (`citizen_verification_codes`: 6 dígitos, 10 min, 5 tentativas, uso único, um ativo por cidadão; código malformado não conta tentativa), o papel `citizen_verifier` (privilegiado) e o módulo "Atendimento" do dashboard (balcão de validação; histórico do admin).
- Unidades de saúde não existem (módulo 09 é esboço). Encaminhamento não existe (módulo 13 é esboço).

## 2. Decisões

1. **Identificação no check-in:** caminho normal por **código** gerado no wpda a partir da triagem ("Cheguei na unidade"); **exceção por CPF** para quem chega sem o código, com motivo obrigatório (≥ 10 caracteres) registrado.
2. **Unidades:** cadastro mínimo `health_units` (nome, tipo, ativa), mantido pelo `municipal_admin`. O atendente escolhe "estou na unidade X"; o navegador lembra a escolha.
3. **Triagens elegíveis:** qualquer triagem web **concluída há 3 dias ou menos** e ainda **sem atendimento**. Um check-in por triagem.
4. **Depois do check-in:** o atendimento fica `open` até o desfecho: **atendido e liberado**, **encaminhado** (para uma unidade cadastrada e/ou descrição curta) ou **saiu sem atendimento**. O cidadão vê no wpda. Nada clínico é guardado.
5. **Quem faz:** o papel `citizen_verifier` (atendente) faz check-in, exceção e desfecho. Qualquer atendente da cidade encerra atendimentos abertos (troca de turno).
6. **Cidadão declarado:** pode fazer check-in. Com código e a caixa "Conferi o documento" marcada, o mesmo passo também **valida o cadastro** (mesma validação do subprojeto 2). Pela exceção, só check-in, sem validar.
7. **Lista de abertos da unidade:** CPF mascarado, hora do check-in, protocolo e **prioridade** da triagem (ampliação deliberada do que o atendente vê no subprojeto 2); sem respostas nem relatório.
8. **Modelo (abordagem A):** o código do balcão ganha finalidade (`verification` | `check_in`) e `triage_id`; o atendimento é uma tabela nova `attendances`; `triages` não muda.
9. **Todo atendimento nasce de uma triagem.** Não há atendimento sem triagem neste subprojeto.

## 3. Dados (banco de cada cidade)

- **`health_units`** (nova): `name`, `kind` (`ubs` | `upa` | `hospital` | `other`), `active` (padrão `true`), timestamps. Nome único na cidade, sem distinguir maiúsculas (`lower(name)`). Unidade inativa some das listas e não aceita check-in nem encaminhamento, mas continua nos atendimentos antigos.
- **`citizen_verification_codes`** (existe) ganha:
  - `purpose` (`verification` | `check_in`, padrão `verification`, CHECK);
  - `triage_id` (FK `triages`, nulo exceto no check-in; CHECK: `purpose = 'check_in'` ⇔ `triage_id IS NOT NULL`).
  - Continua **um código ativo por cidadão**: gerar qualquer código invalida o anterior, de qualquer finalidade.
- **`attendances`** (nova):
  - `triage_id` (FK, **único**), `citizen_id` (FK), `health_unit_id` (FK);
  - `checked_in_by_user_id` (FK users), `checked_in_at`, `check_in_method` (`code` | `cpf_exception`), `exception_reason`;
  - `status` (`open` | `closed`, padrão `open`);
  - `outcome` (`discharged` | `referred` | `left`), `referral_unit_id` (FK `health_units`), `referral_note`, `closed_by_user_id` (FK users), `closed_at`;
  - `created_at`.
  - CHECKs: `cpf_exception` exige `exception_reason` com ≥ 10 caracteres após `btrim` (e `code` exige nulo); `closed` exige `outcome`, `closed_by_user_id` e `closed_at` (e `open` exige os três nulos, mais `referral_*` nulos); `referred` exige `referral_unit_id` ou `referral_note` não vazia; outcomes diferentes de `referred` exigem `referral_*` nulos.
  - Trigger em `db/city_triggers.sql` (protegido por `to_regclass`, como o do subprojeto 2): recusa `DELETE`; recusa `UPDATE` quando `OLD.status = 'closed'`; recusa `UPDATE` que mude colunas do check-in (`id`, `triage_id`, `citizen_id`, `health_unit_id`, `checked_in_*`, `check_in_method`, `exception_reason`, `created_at`).
- **`triages` não muda.**
- **Eventos de trilha:** `attendance.checked_in` (`attendance_id`, `triage_id`, `citizen_id`, `health_unit_id`, `checked_in_by_user_id`, `check_in_method`) e `attendance.closed` (`attendance_id`, `outcome`, `closed_by_user_id`), com `DomainEvents.bind ..., to: []`. Sem CPF, celular, motivo ou descrição no payload.
- As tabelas novas não guardam CPF nem celular; nada entra em `CITY_KEYED_TARGETS`.

## 4. Rotas

```
wpda (sessão do cidadão)
  POST /citizen/triages/:id/check_in_code           → 201 {code, expires_at}
  GET  /citizen/triages   → cada triagem ganha `attendance` (ou null):
       {status, unit_name, checked_in_at, outcome, referral_unit_name, referral_note, closed_at}
       e `check_in_available` (boolean)

dashboard (citizen_verifier)
  GET  /attendance/units                             → [{id, name, kind}] ativas
  POST /attendance/check_ins/lookup   {cpf, code}    → {citizen: {id, cpf_masked, phone_masked, verification_level},
                                                        triage: {id, date, protocol_name, priority}}
  POST /attendance/check_ins          {cpf, code, health_unit_id, document_checked}
                                                     → 201 {attendance: {id, ...}, verified: boolean}
  POST /attendance/check_ins/search   {cpf}          → {triages: [{id, date, protocol_name, priority}]}
  POST /attendance/check_ins/exception {cpf, triage_id, health_unit_id, reason}
                                                     → 201 {attendance: {id, ...}}
  GET  /attendance/units/:id/open                    → [{id, cpf_masked, checked_in_at, protocol_name, priority}]
  POST /attendance/attendances/:id/close {outcome, referral_unit_id?, referral_note?}
                                                     → 200 {attendance: {...}}

dashboard (municipal_admin)
  GET  /attendance/units/all                         → todas, com `active`
  POST /attendance/units              {name, kind}   → 201
  POST /attendance/units/:id          {name, kind}   → 200
  POST /attendance/units/:id/deactivate · POST /attendance/units/:id/activate
```

- `check_in_code`: a triagem precisa ser de um par do celular da sessão (senão 404), `completed`, concluída há ≤ 3 dias e sem atendimento (senão 422 `triage_too_old` / 409 `already_checked_in`).
- `check_ins/lookup` e `check_ins`: o código precisa ter `purpose = 'check_in'`; código de validação responde `invalid_code`. `check_ins` confere e **consome** o código sob lock (como `Citizens::Verify`); se o par é `declared` e `document_checked == true`, cria na mesma transação a validação do subprojeto 2 (mesma tabela, mesmo evento `citizen.verified`) e devolve `verified: true`.
- `check_ins/search` (exceção): só triagens web do CPF, concluídas há ≤ 3 dias e sem atendimento, mais recentes primeiro. Sem celular nem nível do cadastro.
- `check_ins/exception`: a triagem precisa ser do CPF informado e elegível; grava `check_in_method = 'cpf_exception'` e o motivo; não valida o cadastro.
- `close`: qualquer `citizen_verifier`; o atendimento precisa estar `open`; `referral_unit_id` precisa ser unidade ativa.
- O CPF nunca vai na URL. As rotas da cidade usam a sessão da cidade (`Authentication`) dentro de `CityResolution`; toda escrita é JSON. Os limites de 30 a cada 10 minutos por servidor valem para `check_ins/lookup`, `check_ins`, `check_ins/search` e `check_ins/exception` (mesmo limite do balcão de validação).
- **Unidade escolhida:** o dashboard guarda no `localStorage` o id da unidade por usuário; a API recebe `health_unit_id` em cada check-in (não há estado de "unidade atual" no servidor).

## 5. Telas

**wpda — "Minhas triagens"**
- Triagem elegível: botão **"Cheguei na unidade"** → tela do código (a mesma do subprojeto 2), com a instrução "Mostre este código e um documento com foto na recepção da unidade.", contagem e "Gerar outro código".
- Triagem com atendimento: "Em atendimento na *unidade* desde *hh:mm*"; depois do desfecho, "Atendido e liberado", "Encaminhado para *unidade* — *descrição*" (ou só a descrição) ou "Saiu sem atendimento".
- Triagem com mais de 3 dias e sem atendimento: sem botão.

**dashboard — "Atendimento"**
- Topo: **"Unidade: *nome* [trocar]"**. Sem unidade escolhida (ou escolhida e depois desativada), a primeira coisa na tela é a escolha.
- **Check-in:** CPF + código → "Buscar" → cartão com CPF e celular mascarados, nível e triagem (data, protocolo, prioridade); se `declared`, a caixa **"Conferi o documento com foto e o CPF confere"** (opcional: marcada valida junto) → **"Iniciar atendimento"** → confirmação ("Atendimento iniciado" e, se for o caso, "cadastro validado") e **"Próximo atendimento"**.
- **Cidadão sem o código:** CPF → "Buscar triagens" → lista (data, protocolo, prioridade) → escolher → motivo (≥ 10) → **"Iniciar atendimento"**.
- **Atendimentos abertos:** lista da unidade atual (CPF mascarado, hora, protocolo, prioridade), ordenada por prioridade (1 = mais urgente) e depois por chegada; **"Encerrar"** → desfecho: "Atendido e liberado" | "Encaminhado" (unidade de destino entre as ativas e/ou descrição) | "Saiu sem atendimento".
- O balcão de validação e o histórico de validações do subprojeto 2 continuam na mesma tela.

**dashboard — "Unidades"** (só `municipal_admin`; pode ser uma seção do "Atendimento" ou item de menu próprio, decidido no plano): lista (nome, tipo, situação), "Nova unidade", "Editar", "Desativar"/"Reativar".

## 6. Erros

| Situação | Resposta |
|---|---|
| Código errado, vencido, esgotado, malformado; CPF inválido | `invalid_code`, `code_expired`, `code_exhausted`, `invalid_code`, `invalid_cpf` (como no subprojeto 2) |
| Código de validação usado no check-in (ou o contrário) | 422 `invalid_code`, sem dizer por quê |
| Triagem concluída há mais de 3 dias | 422 `triage_too_old` |
| Triagem que já tem atendimento | 409 `already_checked_in`, com `unit_name` e `checked_in_at` |
| Unidade inativa ou inexistente | 422 `invalid_unit` |
| Exceção sem motivo ou com motivo curto | 422 `reason_too_short` |
| Triagem da exceção que não é do CPF informado ou não é elegível | 422 `triage_not_eligible` |
| Encerrar atendimento já encerrado | 409 `already_closed` |
| "Encaminhado" sem destino e sem descrição | 422 `referral_required` |
| Unidade de destino inativa ou inexistente | 422 `invalid_unit` |
| Nome de unidade repetido | 422 `unit_name_taken` |
| Desativar unidade com atendimentos abertos | 409 `unit_has_open_attendances` (revisto na execução: senão os atendimentos ficariam sem como ser encerrados) |
| Sem o papel exigido | 403 `forbidden` |
| Sem sessão / banco da cidade atrasado | 401 / 503 |

## 7. Segurança e LGPD

- O check-in por código segue a regra do balcão: sem o cidadão presente com o código, nada aparece.
- Cada busca da exceção publica `attendance.exception_searched` (`by_user_id`, `result_count`; sem CPF) — revisto na execução, para deixar rastro do acesso.
- A **exceção por CPF** é o único caminho em que o atendente vê triagens de um CPF sem código: exige o papel, conta no limite por servidor, mostra só triagens elegíveis (≤ 3 dias, sem atendimento) e grava método e motivo no atendimento.
- O atendente vê a prioridade (decisão 7); respostas e relatório nunca aparecem no dashboard do atendimento.
- `attendances` é a prova (quem fez o check-in, como, onde, quando; quem encerrou, com qual desfecho); os eventos são trilha.
- Logs: `cpf`, `code` e `reason` já filtrados; `referral_note` entra no filtro.

## 8. Testes

### 8.1 API (RSpec, TDD; request specs com `type: :request`)

- **Unidades:** criar, editar, desativar e reativar só com `municipal_admin`; nome único sem distinguir maiúsculas; unidade inativa fora da lista e recusada em check-in e encaminhamento; atendimentos antigos seguem com o nome.
- **Código com finalidade:** o código de check-in aponta para a triagem certa; código de validação no check-in, e o contrário, dá `invalid_code`; um código novo invalida o anterior de qualquer finalidade; o código de check-in só nasce para triagem própria, concluída há ≤ 3 dias e sem atendimento.
- **`attendances`:** o trigger recusa apagar, encerrar duas vezes e mudar o check-in; os CHECKs de exceção, encerramento e encaminhamento; o índice de um atendimento por triagem.
- **Check-in:** por código abre o atendimento e consome o código; declarado com a caixa → validação + check-in na mesma transação (e o mesmo evento `citizen.verified`); sem a caixa → só check-in; exceção lista só triagens elegíveis, exige motivo e não valida; `triage_too_old`, `already_checked_in`, `triage_not_eligible`; dois atendentes com o mesmo código → um check-in.
- **Encerramento:** os três desfechos; `referral_required`; `already_closed`; lista de abertos ordenada e sem os encerrados.
- **Cidadão:** `GET /citizen/triages` com `attendance` e `check_in_available`; isolamento entre pares como antes (o verificado vê o atendimento das triagens do CPF no histórico completo).
- **Contrato:** check-in e encerramento não mudam triagens, consentimentos, relatórios nem métricas.
- **Guardas:** paridade do schema de cidade; trigger com `to_regclass`; eventos declarados; suíte completa com o worker parado.

### 8.2 dashboard (sem jest-dom) e wpda

- **dashboard:** escolha e troca de unidade (com `localStorage`); check-in por código com e sem a caixa; exceção com motivo; lista de abertos ordenada; encerrar com cada desfecho e "Encaminhado" exigindo destino ou descrição; cadastro de unidades só para o admin; mensagem para cada erro.
- **wpda:** "Cheguei na unidade" só nas triagens elegíveis; tela do código de check-in; situação do atendimento e do encaminhamento.

### 8.3 Verificação em desenvolvimento

Em Curitiba: criar duas unidades; com um cidadão declarado, gerar o código de check-in no wpda; check-in com a caixa → cadastro verificado e atendimento aberto; encerrar como "encaminhado" para a outra unidade e conferir no wpda; repetir pela exceção. O dashboard é exercitado pelo `rails runner` com os mesmos comandos, ou pelo usuário (login de servidor exige senha).

## 9. Fora do escopo

- Encaminhamento em cadeia (check-in automático na unidade de destino).
- Atendimento sem triagem.
- Dado clínico (diagnóstico, prescrição, evolução) — módulo 13.
- Endereço, horário, especialidades e geolocalização das unidades — módulos 09 e 11.
- Vínculo fixo de servidor com unidade.
- Painéis da prefeitura por unidade e métricas de atendimento.
- Status das triagens na lista da validação (previsto no subprojeto 2).
- Agendamento — subprojeto 4.
