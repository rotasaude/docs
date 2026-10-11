# Módulo 20 — Linha do tempo do paciente — design

**Data:** 2026-10-11
**Status:** aprovado (2026-10-11)
**Afeta:**
- `apps/api`: `Timeline::*` (fornecedores, junção, paginação, "a pessoa"), generalização do `ClinicalRecord::Access`, rotas do profissional, do cidadão e da página do PEC; interruptores `patient_timeline` e `pec_external_view`; chave HMAC e endereço do PEC por cidade; índices novos por pessoa e data.
- `apps/dashboard`: aba "Linha do tempo" no painel do paciente.
- `apps/wpda`: área "Meu histórico" (resumo clínico, linha do tempo, documentos).
- `apps/maintenance`: interruptores (mecanismo genérico), chave HMAC e endereço do PEC.
- `contracts`, `apps/signer`, `apps/admin`: nada.

**ADR:** `docs/adr/0035.md` · **Módulo:** `docs/modulos/20--linha-do-tempo.md` · **Pesquisa:** `docs/pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md` (sistemas externos do PEC)

**Fora desta entrega:** exames, vacinas e procedimentos (os módulos que os criarem acrescentam fornecedores); cópia integral do prontuário a pedido; dados vindos do PEC ou da RNDS (sem API de leitura).

## 1. Ponto de partida

- 19a (ADR 0031): paciente por CPF de pares validados; `ClinicalRecord::Access` (in_context, justified, author, administrative); trilha `clinical_record.viewed`.
- 19b/19c/19d: assinatura, documentos com página pública de conferência, receitas e registro de Notificação.
- ADR 0017: par (CPF, celular) do wpda, validação presencial; ADR 0028: modos `off`/`integrated`/`record` e interruptores.
- Pesquisa (frente 1): o PEC não tem API de leitura clínica; a tela de "sistemas externos" abre página externa (iframe ou menu) com `documentoCidadao` (CPF ou CNS), `documentoProfissional`, `CNES`, `pec_timestamp` e assinatura HMAC-SHA256 com chave configurada na instalação.
- Prometido no 19: cidadão vendo o próprio prontuário (ADR 0031) e os documentos no wpda (19c).

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Linha do tempo para o profissional (dashboard) **e** para o cidadão (wpda), no mesmo módulo. |
| 2 | O cidadão vê administrativo + documentos + resumo clínico (problemas, medicamentos, exames solicitados, sinais vitais), sem texto livre de consulta. |
| 3 | Só par validado vê resumo clínico e documentos; par só declarado vê as próprias triagens e agendamentos. |
| 4 | Modo integrado: o que passou pelo Rota Saúde com aviso de histórico parcial **e** página para a tela de "sistemas externos" do PEC. |
| 5 | Pelo PEC, só profissional com vínculo ativo na unidade do CNES. |
| 6 | Abordagem 1: montar na leitura, com fornecedores por módulo (sem tabela de projeção). |

## 3. Interruptores e "a pessoa"

- `patient_timeline` (sem pré-requisito): liga as duas visões. Resumo clínico do cidadão só com `clinical_record` utilizável; documentos só com `clinical_documents`.
- `pec_external_view` (`requires: ["feature:patient_timeline", "record_mode:integrated"]`): liga a página do PEC. Por cidade, gravados só pelo maintenance: `pec_hmac_key` (cifrada com a chave da cidade, nunca devolvida; ação de troca) e `pec_origin` (endereço do PEC, para `frame-ancestors`).
- **A pessoa:** no modo `record`, o paciente do 19a com todos os pares validados ligados (`citizens.patient_id`); no modo integrado, o conjunto dos pares **validados** com o mesmo CPF (cifra determinística). Pares só declarados nunca entram. Para o cidadão no wpda, par só declarado vê só o próprio par.
- `Timeline::Person.for_patient(patient)` / `.for_citizen(citizen)` / `.for_cpf(cpf)` → `{ citizen_ids, patient_id | nil }`.

## 4. Fornecedores e entrada

- `<entry>`: `id` (`<kind>:<source_id>`), `kind`, `occurred_at`, `unit { id, name }`, `professional { name, cbo_code } | null`, `title`, `summary`, `status`, `detail_ref { type, id }`, `citizen_visible`, `citizen_summary`.
- Fornecedores `Timeline::Sources::*` (interface `entries(person:, before:, limit:, kinds:)`):

| kind | Origem | Profissional vê | Cidadão vê |
|---|---|---|---|
| `triage` | 01/03/15 | protocolo, prioridade, desfecho | protocolo, data, prioridade |
| `attendance` | 13 | unidade, chegada, desfecho | unidade, data, desfecho em termos simples |
| `screening` | 18 | cor, destino, CIAP-2 | "acolhimento realizado" |
| `appointment` | 08/17 | data, tipo, situação | data, tipo, situação |
| `consultation` | 19a | CIAP/CID avaliados, condutas, adendos | "consulta com <profissional>" |
| `clinical_document` | 19c/19d | tipo, categoria, modo, situação | tipo, data, situação, PDF e código |
| `controlled_notification` | 19d | tipo, item, número do talão | "receita de controle especial (talão)" |
| `campaign` | 12 | convite, participação | convite, participação |

- Junção: cada fornecedor devolve até `limit` (padrão 50, máximo 100) entradas com `occurred_at < before`; junta, ordena por `occurred_at` desc, `kind`, `id`; devolve `limit` e `next_before` (com desempate por cursor composto `occurred_at|kind|id`). Filtros `kinds[]`, `from`, `to`.
- Índices novos por pessoa (citizen_id/patient_id) e data em cada origem que não tiver.
- **Resumo clínico do cidadão** (`Timeline::CitizenSummary`): problemas ativos (nome), medicamentos em uso, exames solicitados nos últimos 12 meses, última aferição de PA, peso e glicemia com data. Só com paciente e `clinical_record` utilizável.

## 5. Leitura e trilha

- `ClinicalRecord::Access` generalizado para receber a pessoa: `in_context`, `justified`, `administrative` (e `author` para as próprias consultas, no detalhe). Toda leitura da linha do tempo publica `clinical_record.viewed` com `access: "timeline"` (profissional), `"citizen"` (wpda) ou `"pec_external"` (PEC).

## 6. API (resumo; formas no contrato)

- Profissional: `GET /attendance/timeline?patient_id=|attendance_id=|cpf=&before=&limit=&kinds[]=&from=&to=` → `{ person: { display_name, partial: bool }, entries: [<entry>], next_before }`; 403 `out_of_context`/`opening_required` como no 19a; 403 `feature_disabled`.
- Cidadão (sessão do wpda): `GET /citizen/history?before=&limit=` → `{ verified: bool, partial: bool, entries: [<entry cidadão>], next_before }`; `GET /citizen/health_summary` → resumo (404 `not_available` sem prontuário; 403 `not_verified` para par declarado); `GET /citizen/documents/:id/pdf` (só documento da própria pessoa).
- PEC: `GET /pec/timeline?documentoCidadao=&documentoProfissional=&cnes=&pec_timestamp=&assinatura=` → HTML só leitura; `GET /pec/timeline/more?before=` com a sessão curta.

## 7. Página do PEC

- Ordem de conferência: interruptor; HMAC-SHA256 com a chave da cidade (comparação em tempo constante); `pec_timestamp` ≤ 5 min; CNES de unidade ativa; profissional (CPF ou CNS) com vínculo ativo nessa unidade; pessoa com ao menos um par validado. Falha → página simples com motivo genérico e código (nunca qual parâmetro falhou).
- Uso único (assinatura + timestamp guardados até expirar); sessão curta de 15 min só para aquela pessoa e profissional; cookie isolado `SameSite=None; Secure`; `Content-Security-Policy: frame-ancestors <pec_origin>`.
- Mostra a linha do tempo do profissional, só leitura, destacando triagens, escutas, agendamentos e campanhas; aviso "Registros do Rota Saúde — consultas desta cidade estão no PEC". Parâmetros de CPF/CNS mascarados no log.

## 8. Telas

- **Dashboard:** aba "Linha do tempo" no painel do paciente (lista cronológica, filtros, "carregar mais", detalhe nas telas existentes; abertura justificada fora de contexto; aviso de histórico parcial no modo integrado).
- **wpda:** "Meu histórico" — par validado: resumo clínico (quando houver), linha do tempo, documentos; par declarado: triagens e agendamentos + convite para validar.
- **maintenance:** interruptores, chave HMAC (gravar/trocar, nunca mostrar) e endereço do PEC.

## 9. Segurança e LGPD

- Cidadão nunca vê texto clínico livre, CID sem nome, número SNCR nem dado de outra pessoa.
- Chave HMAC cifrada, fora de log; CPF/CNS da chamada do PEC mascarados.
- Eventos só com ids. Leitura com trilha.

## 10. Testes

- **api:** cada fornecedor; junção e paginação (empate, `next_before`, filtros); a pessoa nos dois modos; visibilidade do cidadão por tipo; regras de leitura; página do PEC (válida, inválida, vencida, reenviada, sem vínculo, CNES de outra unidade, sem par validado, `frame-ancestors`); invariantes.
- **dashboard e wpda:** Vitest.
- **Prova no navegador:** profissional nos dois modos; cidadão validado com resumo e download; par declarado com convite; PEC simulado por página de teste que monta a assinatura e abre o iframe.

## 11. Entrega

0. Conferir na documentação oficial dos "sistemas externos" do PEC o formato exato da chamada e da assinatura.
1. `api`. 2. `dashboard`. 3. `wpda`. 4. `maintenance`.

## 12. F-IDs (propostos)

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-20.1 | Interruptores e configuração do PEC | api, maintenance |
| F-20.2 | Fornecedores e linha do tempo da pessoa | api |
| F-20.3 | Linha do tempo do profissional | api, dashboard |
| F-20.4 | Histórico do cidadão (linha do tempo e documentos) | api, wpda |
| F-20.5 | Resumo clínico do cidadão | api, wpda |
| F-20.6 | Página para os "sistemas externos" do PEC | api |

## 13. Riscos e pendências de go-live

- Configuração da URL e da chave em cada PEC real; prova da chamada e do iframe num PEC real.
- Cópia integral do prontuário a pedido.
- Desempenho da junção para pessoas com histórico muito longo (cache por projeção, se preciso).
