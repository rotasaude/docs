# Módulo 20 — Linha do tempo do paciente

- **Estado:** Entregue (F-20.1 a F-20.6 implementadas e provadas no navegador em 2026-10-12; aguardam o `Verified` do usuário). O módulo fica `Fechado` só depois do `Verified` e do gate do PEC real, rotasaude/api#64.
- **Tipo:** Ciclo 2

## Escopo

Linha do tempo da pessoa (ADR 0035), montada na leitura a partir de fornecedores
por módulo (triagem, atendimento, escuta, agenda, consulta, documentos,
Notificação, campanha), com duas visões: o profissional no dashboard (regras de
leitura do 19a) e o cidadão no wpda (administrativo, documentos e resumo
clínico só para par validado). No modo integrado, aviso de histórico parcial e
página para a tela de "sistemas externos" do e-SUS PEC.

**Fora, por enquanto:** exames, vacinas e procedimentos (fornecedores dos
módulos que os criarem); cópia integral do prontuário a pedido.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0035 | Linha do tempo, visões, a pessoa, página do PEC |
| 0031 | Regras de leitura e trilha |
| 0017, 0028 | Par validado; modos e interruptores |

## Funcionalidades

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-20.1 | Interruptores e configuração do PEC | api, maintenance | 0035, 0028 |
| F-20.2 | Fornecedores e linha do tempo da pessoa | api | 0035 |
| F-20.3 | Linha do tempo do profissional | api, dashboard | 0035, 0031 |
| F-20.4 | Histórico do cidadão (linha do tempo e documentos) | api, wpda | 0035, 0017 |
| F-20.5 | Resumo clínico do cidadão | api, wpda | 0035 |
| F-20.6 | Página para os "sistemas externos" do PEC | api | 0035 |

## Critério de fechamento

- F-20.1 a F-20.6 verificadas (`Verified` pelo usuário).
- Prova no navegador (profissional nos dois modos, cidadão validado e declarado, PEC simulado): feita em 2026-10-12.
- Gate de go-live rotasaude/api#64: chamada e iframe provados num PEC real (5.5.x), mais os itens de proxy, log e cadastro listados em "Gates de go-live".
- Runbook (publicação, interruptores, configuração do PEC, URL, verificação, rollback):
  [`operacao/rollout-linha-do-tempo.md`](../operacao/rollout-linha-do-tempo.md).

## Integrações externas

Nenhuma API de leitura (PEC sem leitura clínica; RNDS só em contexto de atendimento). A tela de "sistemas externos" do PEC é ponte possível no modo integrado. Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 19 (a–d).

## Entrega (2026-10-12)

Commits publicados em `origin/main`:

| App | Commit | Alcance |
|---|---|---|
| `api` | `9c7cec8` | 29 commits sobre `2c8e6bb`; suíte 5402/0 |
| `dashboard` | `b4640de` | 13 commits sobre `1107351`; 193 arquivos / 2167 testes |
| `wpda` | `8401032` | 10 commits sobre `20e5e85`; 35 arquivos / 498 testes |
| `maintenance` | `9127bb4` | 4 commits sobre `7db3311`; 29 arquivos / 318 testes |
| `docs` | `98bb9d8` | `pesquisa/2026-10-11-pec-sistemas-externos.md` |

`contracts`, `signer` e `admin`: nada. CI verde nos quatro apps; a do api ficou
verde após rerun (falha de horário em `check_ins_spec`, rotasaude/api#72, que
não é do módulo 20).

Migração de cidade `20261011800002_add_patient_timeline` (**irreversível**):
índices por pessoa e data, `clinical_record_openings.citizen_id` (CHECK de
exatamente um entre `patient_id` e `citizen_id`), `pec_settings` (`pec_origin`,
`hmac_key` cifrada por cidade) e `pec_signature_uses`. Sem migração de
plataforma.

## O que foi entregue

| ID | Entrega | Superfícies |
|---|---|---|
| F-20.1 | Interruptores `patient_timeline` (sem pré-requisito) e `pec_external_view` (exige `patient_timeline`, `record_mode: integrated`, `pec_origin` e `pec_hmac_key`; motivos `patient_timeline_disabled`, `record_mode_not_integrated`, `pec_origin_missing`, `pec_hmac_key_missing`), ambos nascem desligados. GraphQL `cityPecConfig`, `setCityPecOrigin` (origem obrigatória e anulável) e `rotateCityPecHmacKey` (chave + TOTP); evento `city.pec_config_changed` e auditoria `maintenance.city.pec_config_changed`. Quadro "Página do PEC (sistemas externos)" com origem normalizada e chave nunca exibida (dois cliques + TOTP) | api, maintenance |
| F-20.2 | Fornecedores e linha do tempo da pessoa, montada na leitura | api |
| F-20.3 | `GET /attendance/timeline` (prioridade `opening_id` > `patient_id` > `attendance_id`). Aba "Linha do tempo" no painel do paciente (`record`), botão "Linha do tempo" na fila (`integrated`), abertura justificada nos dois modos (no `integrated` abre só a linha do tempo; vale com só `patient_timeline`, pelo par validado, com `patient_id` nulo); pela abertura só a consulta abre; agendamento só abre com `citizen_verifier`; relatório de aberturas ao `municipal_admin` com `clinical_record` OU `patient_timeline` | api, dashboard |
| F-20.4 | `GET /citizen/history?citizen_id=` e `GET /citizen/documents/:id/pdf?citizen_id=` (limite 30 por 10 min); "Meu histórico" no wpda: par validado vê resumo, documentos com PDF e código e linha do tempo; par declarado vê só triagens e agendamentos, com o convite "Validar no posto"; aviso de parcial no `integrated`; botão em "Minhas triagens" só com `patient_timeline`. `GET/POST /citizen/session` devolvem `features` | api, wpda |
| F-20.5 | `GET /citizen/health_summary?citizen_id=` e o resumo no "Meu histórico" | api, wpda |
| F-20.6 | `GET /pec/timeline` (HMAC-SHA256 da URL até `&pec_sign=`, janela de 5 min, uso único, CNES, vínculo, par validado; CSP `frame-ancestors <pec_origin>`; 403 HTML com código) e `GET /pec/timeline/more` (`X-Pec-View-Session`, 15 min, `X-Next-Before`; 401 `pec_session_expired`); rate limit por IP de 120/min e 300/min (429 `too_many_requests`) | api |

Semente de dev (`lib/timeline_crew.rb`): `patient_timeline` ligado; Maringá vira
`integrated` com `pec_external_view`, `pec_origin` `http://pec-sim.localhost:3040`
e chave de dev. O simulador do PEC existe só em `development` (rota e guarda na
ação): `http://pec-sim.localhost:3040/pec-simulator?city=maringa`.

## Provas no navegador (2026-10-12)

- **PEC (simulador, em iframe):** linha do tempo com o aviso; `pec_replayed`,
  `pec_expired`, `pec_professional_not_linked` e `pec_citizen_missing`;
  `frame-ancestors` bloqueia outra origem; log com `[FILTERED]`.
- **dashboard:** `record` (médica, Curitiba) e `integrated` (enfermeira,
  Maringá); abertura justificada nos dois modos; interruptor desligado = 19d.
- **wpda:** par validado (resumo, documentos, PDF, carregar mais); par declarado
  (convite); `integrated` (parcial); interruptor desligado.
- **maintenance:** interruptores, origem normalizada, chave (dois cliques +
  TOTP), auditoria só com o campo.

**Não exercitado no navegador (coberto por testes):** abrir e imprimir
documento e abrir escuta ou pedido no dashboard; "carregar mais" no dashboard;
cancelar no dashboard e baixar no wpda.

## Defeitos achados nas provas (corrigidos)

- O simulador do PEC escolhia o cidadão errado (api `b07aa47`).
- O PDF do cidadão ia sem `citizen_id` (wpda `8401032`); contrato §9 atualizado.
- O relatório de aberturas ficava oculto no `integrated` (dashboard `b4640de`);
  §9 atualizado.

## Gates de go-live

| Item | Bloqueia? |
|---|---|
| rotasaude/api#64 — `pec_external_view` numa cidade real: proxy repassa `Host` e `X-Forwarded-Proto` em `/pec/*`; access log filtra a query de `/pec/*`; nenhum proxy injeta `X-Frame-Options`; cadastro no PEC e chave; prova num PEC real (5.5.x); CNES e CPF/CNS dos profissionais; relógio; rever os tetos do rate limit | **Sim** (só o `pec_external_view`; `patient_timeline` não depende dele) |
| rotasaude/api#65 — caminho de produção para origem e chave do PEC sem console | Alta; hoje se usa o console do Rails |

## Pendências

Board #2 (`mod-20`, `ciclo:02`):

- rotasaude/api#64 — gates de go-live do PEC (Crítica).
- rotasaude/api#65 — caminho de produção para origem e chave do PEC sem console (Alta).
- rotasaude/api#66 — custo das consultas da linha do tempo e da sessão do cidadão (Média).
- rotasaude/api#67 — endurecimento: trilha de pessoa vazia, chave do rate limit sem cidade, CHECK da origem aceita `;`, comentário `city_type` (Baixa).
- rotasaude/api#68 — lacunas de teste (Baixa).
- rotasaude/api#69 — cópia integral do prontuário a pedido (Média, próximo ciclo).
- rotasaude/api#70 — cidadão por CNS na chamada do PEC (Média, próximo ciclo).
- rotasaude/api#71 — exames, procedimentos e vacinas como fornecedores nos módulos 21, 22 e 29 (Baixa, próximo ciclo).
- rotasaude/api#72 — specs de check-in falham entre 23h e 24h de Brasília (módulo 08, Baixa); achado no CI deste módulo, não é do 20.

## Em aberto

- Linha do tempo incompleta no modo integrado (aviso de parcial na tela).

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-11 — Escopo decidido com o usuário; ADR 0035 e spec
  `superpowers/specs/2026-10-11-module-20-patient-timeline-design.md`.
  F-20.1 a F-20.6 criados (board #1, `Not Started`). Módulo passa a `Planejado`.
- 2026-10-12 — **Módulo 20 implementado, publicado e Entregue** (`Verified` do
  usuário e gate do PEC real, rotasaude/api#64, em aberto): api `9c7cec8`,
  dashboard `b4640de`, wpda `8401032`, maintenance `9127bb4`, docs `98bb9d8`
  (pesquisa `pesquisa/2026-10-11-pec-sistemas-externos.md`). Migração de cidade
  `20261011800002` irreversível. Prova no navegador feita (PEC simulado,
  dashboard nos dois modos, wpda validado e declarado, maintenance); três
  defeitos achados e corrigidos (ver acima); pendências api#64 a #71 no board #2.
  Runbook [`operacao/rollout-linha-do-tempo.md`](../operacao/rollout-linha-do-tempo.md).
