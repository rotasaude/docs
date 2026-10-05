# Módulo 15 — Triagem direcionada por perfil

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo

O cidadão deixa de fazer sempre a mesma triagem. Cada cidade oferece um
**catálogo de triagens**, e o wpda mostra a cada pessoa só as que se aplicam a
ela pela idade e pelo sexo. O fim de uma triagem pode **sugerir** outra (ADR
0027).

**Planejado (F-15.1 a F-15.8):**
- perfil do par (CPF, celular): data de nascimento, sexo e identidade de
  gênero opcional; `declared` ou `verified`; obrigatório no wpda depois de
  "para quem é?";
- linguagem de condição com `gte`/`lte` e variáveis reservadas `profile.*`,
  `outcome.*` e `citizen.*`;
- bloco `offer` assinado no protocolo: elegibilidade, título, resumo e
  intervalo de repetição;
- catálogo da cidade (`triage_offers`): pausar, ordenar, restringir (sempre
  somado com E) e período, só pelo `municipal_admin` com step-up;
- catálogo do cidadão no wpda (sugeridas, disponíveis, feitas recentemente) e
  início pelo protocolo escolhido;
- sugestões ao fim da triagem (`suggestions` no protocolo,
  `triage_suggestions`), nunca com resultado urgente;
- construtor visual de condições, simulador de perfil e pré-visualização no
  visual do wpda;
- contadores por protocolo: oferecida, iniciada, concluída, vinda de sugestão.

**Fora, por enquanto:** editor visual completo do protocolo (subprojeto
seguinte), condição de saúde declarada (módulo 23), perfil vindo do CADSUS
(módulo 16), aviso de triagem vencida (módulo 23), recorte por idade e sexo no
Analytics, estimativa de público.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0027 | Catálogo, perfil do par, elegibilidade e sugestões assinadas, restrição da cidade por E, intervalo |
| 0009 | Linguagem de condição, gate, urgência |
| 0016 | Duas assinaturas e step-up |
| 0017 | Par (CPF, celular), sessão por celular, validação presencial |
| 0026 | Exclusão e revogação apagam perfil e sugestões |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `contracts` | Schema `protocols-v1.4.0` (`offer`, `suggestions`, `gte`/`lte`) |
| `api` | Perfil em `citizens`; `triage_offers`, `triage_suggestions`; `Triages::Offer`; `StartTriage` por nome; rotas `/citizen/people/:id/profile`, `/citizen/people/:id/catalog`, `/triage_catalog`, `/authoring/protocols/simulate_offer` |
| `admin` | — |
| `dashboard` | Construtor de condições e simulador no editor de protocolo; aba "Catálogo de triagens"; perfil conferido na validação presencial; contadores |
| `wpda` | Perfil obrigatório, catálogo, "Recomendamos também", "Meu perfil" |

## Pré-requisitos

- Módulo 03 (protocolos assinados) e módulo 06 (validação presencial).
- Módulo 11 (bairro e unidade de referência).
- Por cidade: versão nova do termo de consentimento cobrindo o perfil.

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-15.1 | Perfil do par: nascimento, sexo, identidade de gênero; declarado/conferido; obrigatório no wpda | api, wpda, dashboard | 0027, 0017 |
| F-15.2 | Linguagem de condição com `gte`/`lte` e variáveis `profile.`/`outcome.`/`citizen.` reservadas | contracts, api | 0027, 0009 |
| F-15.3 | Oferta assinada no protocolo: elegibilidade, título, resumo e intervalo de repetição | contracts, api | 0027, 0016 |
| F-15.4 | Catálogo da cidade: pausar, ordenar, restringir (soma com E) e período | api, dashboard | 0027, 0016 |
| F-15.5 | Catálogo do cidadão no wpda e início pelo protocolo escolhido | api, wpda | 0027 |
| F-15.6 | Sugestões ao fim da triagem, pendentes no catálogo (nunca com resultado urgente) | contracts, api, wpda | 0027, 0009 |
| F-15.7 | Construtor visual de condições, simulador de perfil e pré-visualização no visual do wpda | dashboard | 0027 |
| F-15.8 | Contadores por protocolo: oferecida, iniciada, concluída, vinda de sugestão | api, dashboard | 0027, 0025 |

## Riscos herdados

- **Barreira do perfil obrigatório** para quem só quer a triagem de sintomas.
- **Perfil declarado errado** oferece a triagem errada até a validação
  presencial.
- **Sexo como dado sensível de saúde:** cifrado, fora de evento, log, URL e
  Analytics.
- **Termo de consentimento por cidade** fora do repositório: o sistema não
  confere se cobre o perfil.

## Critério de fechamento do módulo

- F-15.1 a F-15.8 verificadas.
- Suíte de invariante (`spec/invariants/triage_catalog_invariants_spec.rb`):
  restrição nunca amplia; `verified` não muda pelo cidadão; nenhum evento, log
  ou URL com nascimento, idade, sexo ou identidade de gênero; urgente não
  sugere; sem autossugestão; versão exata do protocolo; nada cruza pares.
- Runbook de rollout por cidade (termo novo antes do catálogo).

## Histórico

- 2026-10-05 — Escopo decidido com o usuário no brainstorm do Ciclo 2; ADR
  0027 e spec `superpowers/specs/2026-10-05-module-15-triage-catalog-design.md`.
  F-15.1 a F-15.8 criados (board #1, `Not Started`). Módulo nasce
  `Planejado`.
