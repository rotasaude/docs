# Módulo 18 — Acolhimento (escuta inicial) — design

**Data:** 2026-10-07
**Status:** aprovado (2026-10-07)
**Afeta:**
- `contracts`: schema de protocolo `protocols-v1.6.0` (variante `kind: "screening"`, variáveis `vitals.*`, `complaint.ciap2`).
- `apps/api`:
  - cidade: `screenings`, `screening_revisions`, `health_units.screening_scope`, desfechos novos em `attendances`, `appointment_requests.kind = screening`, `ledi_outbox.last_error_codes`;
  - `Screenings::*` (iniciar, abandonar, concluir, reavaliar), `Screenings::RiskSuggestion`, `Screenings::VitalSigns`, fichas `Ledi::Fichas::InitialListening` e `Ledi::Fichas::ScreeningProcedures`, `Ledi::ScreeningFichaJob`, `Ledi::PurgeStalePayloadsJob`;
  - rotas em `/attendance`, `/health_units`, produção.
- `apps/dashboard`: Acolhimento (fila, escuta, reavaliação), fila do profissional com cor, opção da unidade, protocolo `screening` no editor, "fichas não geradas" na Produção.
- `apps/wpda`, `apps/admin`, `apps/maintenance`: nada.

**ADR:** `docs/adr/0030.md` (revisa o 0028 quanto à fila de fichas) · **Módulo:** `docs/modulos/18--acolhimento.md` · **Pendência resolvida:** rotasaude/api#43.

**Fora desta entrega:** escuta domiciliar (módulo 24), escuta visível ao cidadão (módulo 20), painel de espera por cor, CIAP-2 sugerido do texto.

## 1. Ponto de partida

- **`attendances`** (ADR 0018/0019): `status` `waiting` → `in_care` → `closed`; `outcome` ∈ `discharged`, `referred`, `return`, `left`; travas `ck_attendances_closing` (desfecho clínico só de `in_care`; `left` só de `waiting`); origem triagem ou horário (`ck_attendances_origin`).
- **Profissionais** (ADR 0021): vínculo com unidade e CBO; chamada exige vínculo ativo.
- **Exportação** (ADR 0028): `Ledi::Ficha`, `ledi_outbox` (`last_error` string ≤ 500; `payload` apagado só em `accepted`; único `(source_type, source_id, ficha_type)`; `ck_ledi_outbox_rejected_error`), `Platform::Features.usable?(city, :ledi_export)`, terminologias CID-10/CIAP-2/SIGTAP na plataforma, equipes (`health_team_members`), CNES da unidade.
- **Pedido de agendamento** (ADR 0029): `appointment_requests` com origem atendimento ou triagem, tipo, prioridade, prazo.
- **Perfil do par** (ADR 0027): nascimento e sexo.
- **ADR 0026:** revogação mantém o que virou atendimento; exclusão de quem foi atendido fica retida.
- **Pesquisa (frente 1):** escuta inicial vai como Atendimento Individual (MIAI), tipo de atendimento de demanda espontânea; C1 conta só médico e enfermeiro.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Escopo por unidade; padrão só demanda espontânea (`walk_in`), opção `all`. |
| 2 | Escala CAB 28 (4 cores) com sugestão por regras assinadas; decisão humana com justificativa ao mudar. |
| 3 | O acolhimento classifica e define o destino (consulta no dia, agendar, orientação, encaminhar). |
| 4 | Nível superior e técnico/auxiliar de enfermagem; ficha por CBO, com confirmação prévia do layout. |
| 5 | api#43: códigos no lugar de texto, retenção de 90 dias, exclusão limpa a fila. |
| 6 | Queixa com CIAP-2 obrigatório e texto opcional. |
| 7 | Registro próprio (`screenings`), sem estado novo no atendimento. |

## 3. Escuta, sinais vitais e classificação

### 3.1 Dados
- `health_units.screening_scope` (`walk_in`|`all`, padrão `walk_in`).
- `screenings` (cidade): `attendance_id` (único), `status` (`in_progress`|`completed`|`abandoned`), `started_by_user_id`, `professional_link_id`, `cbo_code`, `started_at`, `completed_at`, `current_revision_id`, `destination` (`same_day`|`schedule`|`oriented`|`referred`), `orientation_note` (≤ 500), `appointment_request_id`.
- `screening_revisions` (só acréscimos): `screening_id`, `by_user_id`, `created_at`, `ciap2_code`, `ciap2_release_id`, `complaint_note` (≤ 500), sinais vitais (abaixo), `suggested_color`, `final_color`, `color_change_reason` (≥ 10 quando difere), `rule_protocol_definition_id`, `matched_rules` (índices).
- Sinais vitais (todos opcionais): `systolic`/`diastolic` (mmHg; ambos ou nenhum), `heart_rate` (bpm), `respiratory_rate` (irpm), `temperature_c` (decimal(3,1)), `spo2` (%), `capillary_glucose` (mg/dL) + `glucose_moment` (`fasting`|`postprandial`|`random`), `weight_kg` (decimal(5,2)), `height_cm`, `pain_score` (0–10).
- `Screenings::VitalSigns`: limites de plausibilidade (recusa: ex. sistólica 50–300, diastólica 20–200 e < sistólica, FC 20–250, FR 4–80, temperatura 30–45, SpO2 50–100, glicemia 10–1000, peso 0,5–400, altura 30–250) e faixas de alerta (destaque: ex. sistólica ≥ 140, SpO2 < 95, temperatura ≥ 37,8, glicemia < 70 ou ≥ 200); IMC na leitura.

### 3.2 Comandos
- `Screenings::Start.call(attendance:, by:)`: atendimento `waiting`, unidade exige escuta para ele (escopo), vínculo e CBO permitidos; lock no atendimento; existente `in_progress` → `already_screening`.
- `Screenings::Abandon.call(screening:, by:)` → `abandoned`; volta à fila do acolhimento.
- `Screenings::Complete.call(screening:, revision_params:, destination:, destination_params:, by:)` — §4.
- `Screenings::Reassess.call(screening:, revision_params:, by:)`: só `completed`, destino `same_day`, atendimento `waiting`; nova revisão.
- CBOs permitidos: nível superior da equipe (grupos 2251–2253, 2235, 2234, 2516, 2237, 2236, 2232) e técnico/auxiliar de enfermagem (3222); lista em `config/scheduling/screening_cbos.yml`.

### 3.3 Classificação
- Protocolo assinado `kind: "screening"`: `{ "name", "version", "kind": "screening", "risk_rules": [ { "when": <condição>, "color": "red"|"yellow"|"green"|"blue" } ] }` (até 50 regras). Mesmo ciclo de assinatura, editor e construtor; uma versão `active` por `name`; a cidade usa o nome reservado `acolhimento`.
- Variáveis: `vitals.<campo>`, `vitals.bmi`, `complaint.ciap2`, `profile.age`, `profile.sex`; prefixos reservados `vitals.` e `complaint.`.
- `Screenings::RiskSuggestion.call(revision, profile)` → `{ color: nil|cor, matched: [índices] }`; ordem de gravidade red > yellow > green > blue; puro.
- Semente: rascunho `acolhimento` com regras iniciais baseadas no CAB 28 (ex.: sistólica ≥ 180 ou SpO2 < 90 → red; temperatura ≥ 39 ou glicemia ≥ 300 → yellow).

## 4. Destino e fila

- `same_day`: atendimento segue `waiting`.
- `schedule`: `{ appointment_type_key, priority, due_in_days }` (padrão do prazo pela cor: yellow 7, green 15, blue 30; red sem padrão); cria pedido `kind = screening` (origem `screening_id`) na unidade do atendimento; atendimento fecha `scheduled_from_screening`.
- `oriented`: `orientation_note` obrigatório; fecha `oriented`.
- `referred`: `referral_unit_id` ou `referral_note` (fluxo existente); fecha `referred`; unidade da cidade gera pedido como hoje.
- Trava: `ck_attendances_closing` passa a aceitar `closed` vindo de `waiting` com `outcome` ∈ (`scheduled_from_screening`, `oriented`, `referred`) — a existência de escuta `completed` com o mesmo destino é garantida por trigger (`attendances_screening_close_guard`).
- Fila do profissional: com escuta (`same_day`) por cor (red, yellow, green, blue) e chegada; depois sem escuta (fora do escopo) por horário marcado/chegada. Prioridade da triagem digital só informativa.
- Fila do acolhimento: atendimentos `waiting` que exigem escuta e não têm escuta `completed`, por chegada, com o sinal da prioridade digital.

## 5. Ficha LEDI

- **Tarefa 1 — confirmação do layout:** nos IDLs 8.7.0 (`vendor/ledi/8.7.0`) e na documentação oficial: código de tipo de atendimento da escuta inicial no MIAI; campos de medição; se o MIAI aceita CBO 3222; códigos SIGTAP das aferições (PA, glicemia capilar, antropometria). Resultado em `config/ledi/screening_mapping.yml` com fonte; contradição com este desenho → parar e reportar.
- `Ledi::Fichas::InitialListening` (CBO nível superior) e `Ledi::Fichas::ScreeningProcedures` (CBO 3222, só com aferição); `source_type = "Screening"`.
- Identificação: CNES da unidade, INE da equipe do profissional, CNS/CPF/CBO do profissional, CPF (e CNS) do cidadão, nascimento e sexo do perfil, horário de início/fim, local UBS.
- `Ledi::ScreeningFichaJob`: no fechamento do atendimento e num varredor diário (23h, fuso da cidade) para escutas `completed` de atendimentos ainda abertos; usa a revisão corrente; só enfileira com `usable?(:ledi_export)` e `record_mode != off`.
- Sem identificação: grava `ledi_generation_failures` (`source_type`, `source_id`, `reason_codes`, `created_at`, `resolved_at`) — motivos `unit_without_cnes`, `professional_without_team`, `professional_without_cns`, `citizen_without_birth_date`, `citizen_without_sex`, `unknown_ciap2`; "gerar de novo" (`POST /production/generation_failures/:id/retry`) tenta de novo e resolve.
- Regenerar ficha recusada (corrigida na origem): a linha antiga fica; a nova recebe outro `uuid` e o vínculo `replaces_outbox_id` (o índice único passa a considerar só linhas não `rejected`).
- Modo `integrated`: envia (ADR 0028); runbook alerta para não registrar a escuta também no PEC.

## 6. LGPD (inclui api#43)

- `ledi_outbox.last_error_codes` (jsonb `[{ field, code }]`, campos e códigos de lista fechada; desconhecido → `{ "field": "other", "code": "unknown" }`); `last_error` removido após migração que converte o que der e zera o resto; `ck_ledi_outbox_rejected_error` passa a exigir `last_error_codes` não vazio.
- Resposta crua do PEC só em memória; `Ledi::Outcome` classifica sem persistir texto; nada em log.
- `Ledi::PurgeStalePayloadsJob` (diário, por cidade): `payload = NULL` em `rejected`/`failed` com última tentativa há mais de 90 dias.
- Exclusão confirmada (`Citizens::Erase`): apaga `payload` e `last_error_codes` das linhas da fila cujas fontes são do cidadão.
- Escuta: revogação não apaga (virou atendimento); exclusão de quem foi atendido segue retida.
- Texto livre (`complaint_note`, `color_change_reason`, `orientation_note`) fora de evento, log, Analytics.
- Recepção: na fila só cor, destino, espera.
- Trilha de leitura: `GET` da escuta publica `screening.viewed` (`screening_id`, `user_id`).
- Revisão do ADR 0028 registrando a decisão.

## 7. API (resumo; formatos no contrato entre apps)

- `GET /attendance/units/:id/screening_queue`; `POST /attendance/attendances/:id/screening` (iniciar); `POST /attendance/screenings/:id/abandon`; `POST /attendance/screenings/:id/complete`; `POST /attendance/screenings/:id/reassess`; `GET /attendance/screenings/:id`; `POST /attendance/screenings/suggest` (cor sugerida sem gravar).
- `GET /attendance/units/:id/queue` (existente) ganha `screening: { color, destination, waited_minutes } | null`.
- `POST /health_units/:id` aceita `screening_scope`.
- Produção: `GET /production/generation_failures`, `POST /production/generation_failures/:id/retry`.

## 8. Telas (dashboard)

- Atendimento / Acolhimento: fila, chamar, formulário (CIAP-2 por nome, sinais com alerta, IMC, cor sugerida com motivo, cor final + justificativa, destino), reavaliar.
- Atendimento / fila do profissional: cor, destaque de vermelho, escuta dentro do atendimento chamado; recepção só cor.
- Unidades: `screening_scope`.
- Protocolos: tipo `screening` no editor e no construtor; simulador com sinais vitais; rascunho inicial.
- Produção e-SUS: "Fichas que não puderam ser geradas".

## 9. Testes

- Tarefa 1: mapeamento com fonte; parada em contradição.
- `RiskSuggestion` (tabela de casos), `VitalSigns` (plausibilidade, alerta, IMC, bordas).
- Escuta: lock com threads, abandono, cada destino e desfecho, trava do 13 (fechar de `waiting` sem escuta proibido), reavaliação.
- Filas: ordem por cor e chegada; escopos.
- LEDI: as duas fichas contra os IDLs; geração no fechamento e no varredor; nada com exportação desligada; "não gerada" e "gerar de novo"; regeneração de recusada.
- api#43: `last_error_codes` sem valor (nome/data vindos do PEC); migração de limpeza; purga de 90 dias; exclusão.
- Invariantes (`spec/invariants/screening_invariants_spec.rb`): os do ADR 0030.
- Front (dashboard, relógio fixo).
- Prova no navegador: protocolo ativado; demanda espontânea; PA 185/110 → vermelho no topo; destino agendar → pedido do módulo 17; ficha gerada ou "não gerada".

## 10. Semente de dev

Rascunho `acolhimento` com as regras iniciais, assinado e ativo em Curitiba; enfermeira e técnica de enfermagem com vínculo na UBS da semente; unidade com `screening_scope = walk_in`; dois cidadãos de demanda espontânea com check-in.

## 11. Rollout

1. `contracts` `protocols-v1.6.0`.
2. `api`: migração de cidade (tabelas, desfechos, trava, escopo, `last_error_codes` com limpeza); `VALID_RANGE` → `(1..30)`; jobs novos em `recurring.yml`.
3. `dashboard` depois do `api`.
4. Compatível: sem protocolo ativo não há sugestão; escopo `walk_in` mantém agendados como hoje.

## 12. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-18.1 | Escuta inicial: queixa em CIAP-2, sinais vitais e escopo por unidade | api, dashboard |
| F-18.2 | Classificação de risco CAB 28 com sugestão por protocolo assinado de acolhimento | contracts, api, dashboard |
| F-18.3 | Destino do acolhimento e exceção na trava do módulo 13 | api, dashboard |
| F-18.4 | Filas do acolhimento e do profissional por cor; reavaliação | api, dashboard |
| F-18.5 | Fichas LEDI da escuta por CBO, com confirmação prévia do layout | api |
| F-18.6 | Fichas não geradas por falta de identificação, com motivo e "gerar de novo" | api, dashboard |
| F-18.7 | LGPD na fila de fichas (api#43): códigos, retenção de 90 dias, exclusão | api |
| F-18.8 | Trilha de leitura da escuta | api |

## 13. Riscos

- Confirmação do layout pode mudar o mapeamento CBO → ficha.
- Aceite real do PEC ainda não observado (api#41): passo de go-live.
- Dupla contagem no modo integrado depende de orientação operacional.
- Regras iniciais de cor são ponto de partida; a cidade precisa revisar e assinar.
