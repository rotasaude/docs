# Módulo 10 — Profissionais (F-10.1 a F-10.5) — design

**Data:** 2026-09-27
**Status:** aprovado em conversa (2026-09-27), aguardando revisão do texto
**Afeta:**
- `apps/api`:
  - banco de cada cidade: `professionals`, `professional_links` e `professional_shifts` novas, extensão `btree_gist`;
  - lista CBO da saúde versionada (`config/professionals/cbo_saude.yml`);
  - rotas `/professionals*`, `/links/*`, `/shifts/*`, `/cbo`;
  - regra clínica em `Attendances::Call`, `CallNext` e `Close`;
  - `GET /setup/memberships` ganha `professional_status`;
  - semente de dev (unidades e profissionais).
- `apps/dashboard`: módulo "Profissionais" (admin), "Meu perfil" (profissional), etiquetas na Equipe, Atendimento restrito ao vínculo.

**ADR:** `docs/adr/0021.md` (emendado por esta entrega, §8) · **Módulo:** `docs/modulos/10--profissionais.md`

**Fora desta entrega:** F-10.6 (CNES) e F-10.7 (escalonamento do alerta urgente), em aberto no ADR 0021. Semana-padrão gerando turnos. Agenda de vagas consumindo turnos (módulo 08).

## 1. Ponto de partida

- **ADR 0019:** o papel `health_professional` chama (`waiting → in_care`) e registra o desfecho clínico em **qualquer** unidade. A recepção (`citizen_verifier`) marca "saiu sem atendimento" (`left`).
- **Código hoje:**
  - `Attendances::Call` trava o atendimento e confere a unidade;
  - `CallNext` tenta até 3 vezes;
  - `Close` separa `left` de desfecho clínico;
  - a autorização é só por papel (`CitizenVerificationPolicy#care?`, `AttendanceAccess#require_professional`);
  - a fila mostra quem chamou pelo **e-mail** (`called_by_name: email_address`).
- **Módulo 09:** `HealthUnit.lock_active!` (`FOR SHARE`) roda dentro da transação de `CheckIn`, `CheckInByException` e `Close`. **Continua onde está.**
- **Semente de dev:** `SignatureCrew` cria `profissional@<cidade>.demo` (papel `health_professional`) e `recepcao@<cidade>.demo` (papel `citizen_verifier`), com TOTP fixo. **Não cria unidades.**

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| D1 | Escopo: F-10.1 a F-10.5. |
| D2 | Sem backfill. Em staging e produção, um `health_professional` sem perfil e vínculo deixa de chamar até o admin cadastrar; a Equipe e o painel de pendência mostram quem falta. Em dev, a semente cria profissionais fictícios **com forma real**. |
| D3 | O `municipal_admin` cria e edita tudo. O profissional vê o próprio perfil, vínculos e turnos só para leitura, e edita apenas `professional_name`, `phone` e `contact_email`. |
| D4 | Contato = telefone profissional e e-mail de contato, ambos opcionais e cifrados com a chave da cidade. O e-mail de login continua em `users`. |
| D5 | Step-up (TOTP recente, `MfaStepUp`) para **abrir e encerrar vínculo**. Perfil e turnos sem step-up. |
| D6 | Turno gravado como instantes (`starts_at`, `ends_at`). A tela pede data e horas; fim ≤ início = dia seguinte. Duração máxima de 24h, sem sobreposição por profissional em qualquer unidade. |
| D7 | Regra clínica num único ponto de domínio, checada dentro da transação dos comandos, com trava no vínculo (abordagem 1). |
| D8 | Encerrar vínculo cancela, na mesma transação, os turnos dele que ainda não começaram (motivo "vínculo encerrado"). Turnos passados e o turno em curso ficam. |
| D9 | Ao abrir vínculo, se o CBO declara um conselho, o conselho do perfil tem de ser esse. CBO sem conselho aceita qualquer perfil. Quem tem dois registros fica com um só (limite aceito). |

## 3. Dados

Uma migração em `db/city_migrate`, só de expansão e reversível.

### 3.1 `professionals`

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | uuid | |
| `user_id` | uuid | NOT NULL, UNIQUE, FK `users` |
| `professional_name` | string | NOT NULL, sem espaços nas pontas, não vazio |
| `council` | string | NOT NULL; um de `Professional::COUNCILS` (CRM, COREN, CRO, CRF, CRP, CREFITO, CRN, CRFa, CRESS, CRBM, CREF, CRMV) |
| `council_state` | string(2) | NOT NULL; uma das 27 UFs |
| `registration_number` | string | NOT NULL; só dígitos, 1 a 10; em claro (dado público dos conselhos) |
| `cns` | string | NOT NULL; 15 dígitos com dígito verificador válido; cifrado **determinístico** (`CityDeterministicKeyProvider`) |
| `phone` | string | opcional; formato brasileiro (DDD + 8 ou 9 dígitos); cifrado |
| `contact_email` | string | opcional; formato de e-mail; cifrado |
| `created_at`, `updated_at` | | |

- Índices: UNIQUE (`council`, `council_state`, `registration_number`); UNIQUE (`cns`), que funciona porque a cifra é determinística.
- Tabela editável. Toda edição publica `professional.profile_updated` com os **nomes** dos campos alterados, nunca os valores.
- Validação do CNS: 15 dígitos, primeiro dígito em 1, 2, 7, 8 ou 9, e soma ponderada (pesos 15 a 1) múltipla de 11. Vale para as duas famílias, porque o número definitivo (1 ou 2, derivado do PIS) é construído para fechar essa soma. `Professionals::Cns` é a fonte única de `valid?` e do gerador usado pela semente.

### 3.2 `professional_links`

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | uuid | |
| `professional_id` | uuid | NOT NULL, FK |
| `health_unit_id` | uuid | NOT NULL, FK |
| `cbo_code` | string(6) | NOT NULL; código da lista CBO vigente (validação no modelo) |
| `started_at` | timestamptz | NOT NULL; instante da ação |
| `started_by_user_id` | uuid | NOT NULL, FK `users` |
| `ended_at` | timestamptz | nulo enquanto ativo |
| `ended_by_user_id` | uuid | nulo enquanto ativo |
| `created_at` | | |

- CHECK: `ended_at IS NULL OR ended_at >= started_at`; `ended_at` e `ended_by_user_id` nulos juntos ou preenchidos juntos.
- Índice único parcial: (`professional_id`, `health_unit_id`, `cbo_code`) WHERE `ended_at IS NULL`.
- **Trigger de só-acréscimo** (`professional_links_guard`): DELETE proibido. UPDATE só é aceito quando `ended_at` e `ended_by_user_id` passam de NULL para preenchidos e nenhuma outra coluna muda.
- `started_at` e `ended_at` são sempre `Time.current` no comando: sem data retroativa.
- Não abre vínculo em unidade inativa.

### 3.3 `professional_shifts`

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | uuid | |
| `professional_link_id` | uuid | NOT NULL, FK |
| `professional_id` | uuid | NOT NULL, FK; igual ao do vínculo (denormalizado para a EXCLUDE; conferido pelo trigger no INSERT) |
| `starts_at`, `ends_at` | timestamptz | NOT NULL |
| `created_by_user_id` | uuid | NOT NULL |
| `cancelled_at`, `cancelled_by_user_id`, `cancel_reason` | | nulos enquanto válido; preenchidos juntos |
| `created_at` | | |

- CHECK: `ends_at > starts_at` e `ends_at - starts_at <= interval '24 hours'`.
- `EXCLUDE USING gist (professional_id WITH =, tstzrange(starts_at, ends_at) WITH &&) WHERE (cancelled_at IS NULL)`. Precisa de `CREATE EXTENSION IF NOT EXISTS btree_gist`, que é trusted no PG ≥ 13. **A fatia 1 confirma que o papel que migra o banco da cidade consegue criá-la**, em dev e no banco de teste; se não conseguir, a extensão entra no provisionamento da cidade.
- **Trigger de só-acréscimo** (`professional_shifts_guard`): DELETE proibido. UPDATE só preenche o trio de cancelamento, uma vez, a partir de NULL. No INSERT, confere que `professional_id` é o do vínculo.
- O turno começa em ou depois do `started_at` do vínculo. Turno no passado é permitido (registro de quem estava escalado).
- Fuso: o da aplicação (`America/Sao_Paulo`). Não há fuso por cidade hoje.

### 3.4 Lista CBO

`config/professionals/cbo_saude.yml`, versionada no api e igual para todas as cidades. Cada entrada tem `code` (6 dígitos), `title`, `council` (ou `null`) e `deprecated` (padrão `false`).

- Conteúdo inicial: as ocupações de saúde de uso corrente no CNES. Médicos (famílias 2231 e 2251 a 2253, com clínico 225125, pediatra 225124, médico de família 225142, ginecologista 225250, psiquiatra 225133 entre outros), enfermagem (223505 enfermeiro, 322205 técnico, 322230 auxiliar), odontologia (223208 cirurgião-dentista clínico, 322405 técnico em saúde bucal), farmácia (223405), psicologia (251510), fisioterapia (223605), nutrição (223710), fonoaudiologia (223810), serviço social (251605) e 515105 agente comunitário de saúde (sem conselho).
- `Professionals::Cbo` carrega o arquivo uma vez e expõe `find(code)`, `active` e `council_for(code)`.
- Código nunca sai do arquivo; quando cai em desuso, vira `deprecated: true`. Um código `deprecated` não abre vínculo novo, mas continua válido nos vínculos que já existem.

### 3.5 Eventos

`DomainEvents` da cidade, **só ids**: `professional.created`, `professional.profile_updated` (com `fields: [...]`), `professional.linked`, `professional.unlinked` (com `cancelled_shift_ids`), `professional.shift_scheduled` e `professional.shift_cancelled`. Nenhum payload leva CNS, número de registro, telefone, e-mail ou nome.

## 4. API

### 4.1 Rotas

| Rota | Quem | Comportamento |
|---|---|---|
| `GET /professionals` | admin | Perfis com vínculos ativos e encerrados (unidade, CBO e título, início, fim, quem abriu e quem encerrou). CNS mascarado (`*** **** **** 1234`); o CNS inteiro só aparece em `GET /professionals/:id` |
| `GET /professionals/:id` | admin | Perfil completo, com CNS, telefone e e-mail decifrados |
| `GET /professionals/pending` | admin | Usuários com `health_professional` ativo e `professional_status != ok` |
| `POST /professionals` | admin | Cria o perfil (`user_id` + campos). Sem o papel ativo → 422 `missing_role`; perfil já existente → 409 `already_exists` |
| `POST /professionals/:id` | admin | Edita qualquer campo do perfil (menos `user_id`) |
| `GET /professionals/me` | quem tem perfil | Próprio perfil completo, vínculos ativos e turnos válidos dos próximos 14 dias. Sem perfil → 404 `no_profile` |
| `POST /professionals/me` | quem tem perfil | Aceita só `professional_name`, `phone`, `contact_email`; qualquer outra chave → 422 `field_not_editable` com a lista |
| `POST /professionals/:id/links` | admin + step-up | `{health_unit_id, cbo_code}`. Erros: `mfa_required` (401), `invalid_unit`, `invalid_cbo`, `council_mismatch` (422), `already_linked` (409) |
| `POST /links/:id/end` | admin + step-up | Encerra e cancela os turnos futuros. Já encerrado → 409 `already_ended` |
| `GET /professionals/:id/shifts?from=&to=` | admin | Turnos de todos os vínculos no intervalo (padrão: hoje + 14 dias; máximo de 62 dias), válidos e cancelados |
| `POST /links/:id/shifts` | admin | `{starts_at, ends_at}` em ISO 8601. Erros: `link_ended` (409), `shift_overlap` (409, com o turno em conflito: unidade, início e fim), `invalid_shift` (422: fim antes do início, mais de 24h, antes do início do vínculo) |
| `POST /shifts/:id/cancel` | admin | `{reason}` obrigatório, até 200 caracteres. Já cancelado → 409 `already_cancelled` |
| `GET /cbo` | admin | Lista sem os `deprecated` |

"admin" = `municipal_admin` ativo. Qualquer outro papel nas rotas de admin → 403.

### 4.2 Comandos (`app/commands/professionals/`)

Todos devolvem `Result` e publicam o evento na mesma transação.

- `Create`: exige o papel ativo, valida o perfil e publica `professional.created`.
- `UpdateProfile` (admin) e `UpdateOwnProfile` (lista fechada de campos): publicam `professional.profile_updated` só quando algo mudou.
- `OpenLink`, que confere, nesta ordem:
  - unidade ativa;
  - CBO vigente;
  - coerência com o conselho (D9);
  - sem vínculo ativo igual.
  A corrida no índice parcial vira `already_linked`.
- `EndLink`: trava o vínculo (`FOR UPDATE`), confere `ended_at IS NULL`, grava o fim, cancela os turnos com `starts_at > now` e publica `professional.unlinked`.
- `ScheduleShift`: trava o vínculo (`FOR SHARE`), confere que está ativo e insere. A violação da EXCLUDE vira `shift_overlap`, com o turno em conflito.
- `CancelShift`: trava o turno e grava o cancelamento.

### 4.3 Regra clínica (F-10.5)

`Professionals::ClinicalAuthorization.check!(user:, health_unit_id:)`, chamada **dentro** de uma transação já aberta:

1. papel `health_professional` ativo para o usuário; senão, `Result.fail(:missing_role)`;
2. `professional_links` ativo do perfil desse usuário com essa unidade, em qualquer CBO, travado com `FOR SHARE`; senão, `Result.fail(:missing_link)`;
3. `:ok`. **Nenhuma consulta a `professional_shifts`.**

| Comando | Onde entra |
|---|---|
| `Attendances::Call` | depois de `attendance.lock!` e da conferência de unidade |
| `Attendances::Close` | só quando `outcome != "left"`, depois de `attendance.lock!`; `left` segue sem vínculo |
| `Attendances::CallNext` | uma vez antes do laço de tentativas (atalho); o `Call` rechecar dentro é o que garante |

Os controllers mantêm a checagem de papel já existente, que responde cedo. As falhas `missing_role` e `missing_link` saem como **403** com `{"error": "<motivo>"}`, pela tabela de status do `AttendancesController`.

A trava do vínculo com `FOR SHARE` ordena o encerramento contra a chamada: se o `EndLink` vem primeiro, a chamada recebe `missing_link`; se a chamada vem primeiro, o `EndLink` espera o commit dela.

### 4.4 Efeitos em rotas existentes

- `queue_json` e `attendance_json`: `called_by_name` passa a mostrar `professional.professional_name`, caindo para o e-mail sem perfil; `closed_by_name` entra com a mesma regra. Consulta com `includes` para não gerar N+1.
- `GET /setup/memberships`: para usuário com `health_professional` ativo, `professional_status` = `missing_profile` | `missing_link` | `ok`.
- Revogar o papel ou desativar o usuário não altera perfil, vínculos nem turnos.

## 5. Dashboard

- **Navegação:**
  - "Profissionais" entra no grupo Equipe, visível só para `municipal_admin`;
  - "Meu perfil" entra no grupo Conta, para quem tem o papel `health_professional`, e fica fora para operador.
- **Profissionais — lista:**
  - colunas: nome, conselho/UF/número, vínculos ativos (unidade · título do CBO) e etiqueta de pendência;
  - no topo, o painel "Com papel, sem cadastro completo" (`/professionals/pending`), com o botão "Cadastrar perfil" ou "Vincular".
- **Profissionais — ficha:**
  - **Perfil:**
    - formulário com todos os campos;
    - CNS validado na tela com a mesma regra do api (portada, testada contra os mesmos casos);
    - CNS mascarado depois de salvo.
  - **Vínculos:**
    - tabela de ativos e encerrados;
    - "Abrir vínculo" tem seletor de unidades ativas e seletor de CBO com busca por código ou título;
    - "Encerrar" mostra quantos turnos futuros serão cancelados;
    - as duas ações passam por `SensitiveAction` + `useStepUp`.
  - **Turnos:**
    - por vínculo, uma janela de 14 dias com navegação por semana;
    - "Lançar turno" pede data, hora de início e hora de fim. Quando fim ≤ início, mostra "termina em DD/MM às HH:MM" antes de salvar e envia os dois instantes;
    - "Cancelar" pede o motivo;
    - `shift_overlap` nomeia o turno em conflito.
- **Equipe:** na linha de quem tem `health_professional` com `professional_status != ok`, a etiqueta "sem perfil" ou "sem vínculo" leva à ficha.
- **Meu perfil:**
  - leitura de conselho, registro, CNS mascarado, vínculos ativos e próximos turnos;
  - edição de nome profissional, telefone e e-mail de contato;
  - sem perfil: "Seu cadastro profissional ainda não foi feito. Fale com a administração da cidade."
- **Atendimento:**
  - `canCare` = papel **e** vínculo ativo com a unidade selecionada, a partir de `/professionals/me`;
  - sem vínculo, somem Chamar, Chamar próximo e o desfecho clínico, e aparece "Você não tem vínculo com esta unidade"; "Saiu sem atendimento" segue para a recepção;
  - 403 com `missing_link` ou `missing_role` mostra a mensagem e invalida a query de `/professionals/me`;
  - a fila mostra `called_by_name`.

## 6. Semente de dev

Dev é fictício, mas imita o real. Tudo passa pelos comandos do domínio, para as validações rodarem. A semente é idempotente e roda por cidade, depois do `SignatureCrew`.

- **Unidades**, criadas se não existirem, achadas pelo nome: "UBS Jardim das Flores" (`ubs`), "UBS Vila Esperança" (`ubs`) e "UPA 24h Centro" (`upa`).
- **Profissionais** (UF tirada do cadastro da cidade; nomes brasileiros fictícios; registros no formato do conselho; CNS válido, gerado de forma determinística a partir de cidade e conta):

| Conta | Perfil | Vínculos | Turnos (relativos ao dia do seed) |
|---|---|---|---|
| `profissional@` (existente) | médica, CRM | UBS Jardim das Flores como 225125; UPA como 225124 | 07–13 na UBS nos próximos dias úteis; um 19h–07h na UPA |
| `enfermeira@` | COREN | UBS Jardim das Flores como 223505; UBS Vila Esperança como 223505, **encerrado** | 07–19 na UBS Jardim das Flores |
| `tecnico@` | COREN | UPA como 322205 | um plantão 07h–07h (24h) |
| `novato@` | papel `health_professional`, sem perfil | — | — |

- As contas novas ganham TOTP fixo com override por env, no padrão de `SignatureCrew::MEMBERS`.
- Idempotência:
  - perfil achado pelo usuário;
  - vínculo pelo par ativo (profissional, unidade, CBO);
  - o vínculo encerrado só é criado se não existir nenhum vínculo daquele par;
  - turnos só lançados se o vínculo não tiver turno futuro válido.
- `recepcao@` continua sem perfil: é quem marca `left`.

## 7. Testes

### 7.1 Invariantes: `spec/invariants/professional_invariants_spec.rb`

Cada linha tem a spec e a mutação que precisa deixá-la vermelha. A evidência (vermelho → verde) vai no relatório da fatia.

| Invariante | Prova | Mutação |
|---|---|---|
| Perfil 1:1 com o usuário | 2º perfil do mesmo usuário falha **no banco** | remover o UNIQUE de `user_id` |
| Perfil só com o papel | `Create` sem o papel → `missing_role` | tirar a checagem |
| Vínculo só por acréscimo | por SQL: UPDATE de `started_at`, `health_unit_id` e `cbo_code`, 2º `ended_at` e DELETE falham | desligar `professional_links_guard` |
| Um vínculo ativo por (profissional, unidade, CBO) | 2º insert ativo igual falha no banco | remover o índice parcial |
| Turno só por acréscimo, sem sobreposição, ≤ 24h | por SQL contra o trigger, a EXCLUDE e o CHECK | remover cada um |
| Sem turno em vínculo encerrado | `ScheduleShift` recusa `link_ended`; `EndLink` cancela os futuros e mantém o passado e o em curso | tirar o cancelamento do `EndLink` |
| Só o `municipal_admin` cadastra | request spec de toda rota de escrita com cada outro papel → 403 | afrouxar o filtro |
| Papel + vínculo com a unidade do atendimento | sem papel → `missing_role`; sem vínculo, vínculo em outra unidade ou vínculo encerrado → `missing_link`; `left` pela recepção sem vínculo → ok | tirar o `check!` de `Call` e de `Close` |
| Turno nunca bloqueia ato clínico | vinculado sem turno, ou com turno cancelado, chama e fecha | fazer a regra consultar turnos |
| Eventos sem dado sensível | payload de cada `professional.*` sem CNS, registro, telefone, e-mail ou nome | pôr o `cns` no payload |

### 7.2 Demais

- Unidade: `Professionals::Cns` (válidos e inválidos das duas famílias), `Professionals::Cbo`, validações de modelo, cada comando.
- Request specs de todas as rotas da §4.1, incluindo step-up (`mfa_required` sem TOTP recente) e `field_not_editable`.
- Concorrência (threads, `pop(timeout:)`, limpeza no `after`): "encerrar vínculo × chamar" e "encerrar vínculo × lançar turno".
- Regressão:
  - `appointment_invariants_spec.rb` e `health_unit_invariants_spec.rb` verdes;
  - specs de `Call`, `CallNext` e `Close` ganham perfil e vínculo no setup;
  - `HealthUnit.lock_active!` intacto.
- Semente: spec que roda duas vezes e confere contagens estáveis e CNS válidos.
- Suíte completa com o worker parado. Nenhum teste com data fixa contra o relógio real.
- Dashboard (vitest + Testing Library): lista, ficha, "Meu perfil", etiquetas da Equipe, `canCare` por vínculo, mensagem de 403. Turno que atravessa a meia-noite com `vi.setSystemTime`.

## 8. Emenda ao ADR 0021

Entra com a entrega, fechando itens de "Em aberto" e precisando o texto:

- **Autoedição:** o profissional vê o próprio perfil, vínculos e turnos e edita apenas o nome profissional e o contato (telefone e e-mail de contato). Conselho, registro, CNS, vínculos e turnos continuam com o admin, porque a prefeitura confere esses dados.
- **Turnos como instantes:** "data, hora de início e de fim" passa a ser `starts_at`/`ends_at`, até 24h e sem sobreposição por profissional; a "data" é a do início.
- **Encerrar vínculo cancela os turnos futuros** dele; turnos passados e em curso ficam.
- **CBO coerente com o conselho** do perfil, quando o CBO declara um.
- **Step-up** para abrir e encerrar vínculo.
- **Recusas nomeadas:** `missing_role` e `missing_link`.

## 9. Fatias e entrega

| Fatia | Repo | Conteúdo | F-IDs |
|---|---|---|---|
| 1 | api | migração (três tabelas, extensão, triggers), `Professional`, `Cns`, comandos e rotas de perfil (`me` e `pending`), eventos de perfil | F-10.1 |
| 2 | api | lista CBO, `OpenLink`/`EndLink` com step-up, coerência com o conselho, rotas de vínculo | F-10.2, F-10.3 |
| 3 | api | `ScheduleShift`/`CancelShift`, cancelamento no `EndLink`, rotas de turno | F-10.4 |
| 4 | api | `ClinicalAuthorization` em `Call`/`CallNext`/`Close`, `called_by_name`/`closed_by_name`, `professional_status`, semente de dev, suíte de invariantes completa | F-10.5 |
| 5 | dashboard | Profissionais, Meu perfil, Equipe, Atendimento | F-10.1–F-10.5 (telas) |

- As três tabelas nascem juntas na fatia 1, numa migração só, para não haver três migrações de cidade para uma única entrega. As fatias 2 e 3 ativam o comportamento em cima delas.
- Um worktree por repo a partir de `origin/main`; TDD por subagentes; revisão do diff de produção por subagente antes de cada merge.
- **Ordem de deploy:** api (fatias 1–4) antes do dashboard. A fatia 4 muda o comportamento: em staging e produção, perfis e vínculos são cadastrados **antes** de publicar a imagem com ela (D2).
- Merge, push, board, docs e estado do módulo (Planejado → Em andamento → Entregue) só com autorização explícita, uma etapa de cada vez.

## 10. Riscos

- **`btree_gist` sem permissão** no papel de migração da cidade. Mitigação: conferido na fatia 1; se faltar, a extensão vai para o provisionamento.
- **Cidade sem cadastro quando a fatia 4 entra**, e ninguém chama. Mitigação: painel de pendência, etiqueta na Equipe e passo de rollout explícito.
- **Lista CBO incompleta** para alguma ocupação local. Mitigação: é dado versionado; acrescentar um código é um commit, sem migração.
- **Trabalho de lançar turno a turno.** Custo aceito no ADR 0021; a semana-padrão pode vir depois, gerando turnos.
