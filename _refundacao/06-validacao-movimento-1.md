# Validação do Movimento 1 — 2026-06-23

- **Etapa:** 6 (validação documental final) · **Modo:** read-only (audita, não corrige)
- **Escopo auditado:** corpus v2 completo em `<destino>/docs/` + `.github/`
- **Método:** cruzamento do destilado (Etapa 0) e do mapa (Etapa 1) contra os 15
  ADRs v2 (Etapa 3), o de-para (Etapa 4) e a reconciliação de docs (Etapa 5),
  com verificação por grep onde automatizável.

---

## 6.1 Integridade referencial

- **Refs v1 órfãs:** **0**. Nenhum doc em `docs/` cita número v1-only (0016–0026)
  fora do `DE-PARA.md` (onde é legítimo). Verificado por grep.
- **Links quebrados:** **0**. Varredura de todos os links markdown internos
  (ADR→ADR, módulo→ADR, ciclo→ADR, índice→ADR) — todos resolvem para arquivo
  existente.
- **DE-PARA completo:** **Sim**. 26 origens v1 (0001–0025 + 0026/Anexo A) + a nota
  de custódia (Anexo B) → 15 ADRs v2, nas duas direções, sem buraco. Splits (0011,
  0020) marcados.

## 6.2 Invariantes (a checagem crítica)

As 15 invariantes do destilado (Etapa 0, seção 3) — decisões de segurança clínica,
LGPD e isolamento — todas presentes no ADR v2 esperado. Verificado por grep de
frase-âncora.

| # | Invariante (destilado) | ADR v2 | Presente? |
|---|---|---|---|
| 1 | Exactly-once por consumidor (`(event_id, consumer)` único) | 0005 | **sim** |
| 2 | HMAC antes de qualquer INSERT; `secure_compare` | 0007 | **sim** |
| 3 | Sem `consented` não alimenta triagem; revogação aborta em curso | 0008 | **sim** |
| 4 | RLS falha fechada: sem tenant, query nega (não libera) | 0003 | **sim** |
| 5 | Cross-tenant só explícito via `rota_admin` | 0003 | **sim** |
| 6 | Criptografia em repouso de PII/saúde (AR Encryption) | 0013 | **sim** |
| 7 | `domain_events` imutável (só `published_at` muda) | 0004 | **sim** |
| 8 | `ReportSnapshot` individual imutável (prova) | 0010 | **sim** |
| 9 | Control plane RLS-exempt protegido por RBAC | 0003 / 0012 | **sim** |
| 10 | MFA: operador no login; publisher em step-up | 0011 | **sim** |
| 11 | Append-only: `consents`, `memberships`, `users` | 0008 / 0012 | **sim** |
| 12 | Publicação de protocolo deliberada e rastreável (step-up + `protocol.published`) | 0009 / 0012 | **sim** |
| 13 | Isolamento de SLA clínico: pesado nunca atrasa alerta urgente | 0006 | **sim** |
| 14 | HTTP fora do lock; idempotency key no provedor | 0005 | **sim** |
| 15 | `solid_queue_*` e control plane **não** recebem RLS (exceção deliberada) | 0003 | **sim** |

**Invariantes perdidas: 0.** Nenhuma decisão de segurança sumiu na fusão. (Várias
foram reformuladas — ex.: "RLS isola por tenant" → "nenhuma query de domínio roda
sem tenant; falha fechada" — mas é a mesma invariante reescrita, não perdida.)

## 6.3 Contratos

Todo schema/contrato do destilado aparece no ADR v2 correspondente, no estado final.

| Contrato | ADR v2 | Presente / estado final? |
|---|---|---|
| Tabela `domain_events` (com `municipality_id`, `aggregate_*`) | 0004 | **sim** (forma autoritativa: publish carimba tenant + insere no mesmo átomo) |
| `DomainEvents.publish` (assinatura) + bindings | 0004 | **sim** |
| `Result` + tabela de commands (`CompleteTriage`, `GiveConsent`, `RevokeConsent`) | 0004 | **sim** (EN) |
| Tabela `processed_events` (`(event_id, consumer)`) | 0005 | **sim** |
| Policy SQL RLS + dois papéis + `with_tenant` | 0003 | **sim** |
| `municipality_channels` + shape do `InboundMessage` | 0007 | **sim** |
| Máquina de estados da `Conversation` + tabela `Consent` | 0008 | **sim** |
| `Outcome` (status/tier/priority/trail) | 0009 | **sim** |
| `protocol_definitions` + façade + scoring (weighted/decision_table) | 0009 | **sim** (estado final do 0016, não o esboço do 0013-v1) |
| `ReportSnapshot` + `DashboardMetric` | 0010 | **sim** |
| `users`/`sessions`/`identities` + MFA | 0011 | **sim** |
| `memberships` + tabela de roles | 0012 | **sim** |
| `municipalities` + `ProvisionMunicipality` + `consent_terms` + `alert_recipients` + chaves AR Encryption | 0013 | **sim** |
| SemVer por domínio + expand/contract + tolerância do consumidor | 0015 | **sim** |

**Contratos perdidos: 0.**

## 6.4 Em aberto

- **Consolidação correta:** os "Em aberto" dos 15 ADRs estão consolidados no
  `adr/README.md`, sem duplicata, agrupados por área (Arquitetura / Confiabilidade
  clínica / Operação e custódia / Identidade e LGPD).
- **Resolvidos ressuscitados: 0.** Verificado que **não** reaparecem: `packages/ui`,
  `packages/types`, "design system compartilhado", "auth ausente", política de
  versionamento de `contracts` (resolvida pelo 0015), custódia §5.4 (resolvida pelo
  0013).
- **Perda conhecida (Etapa 1, aceita pelo autor):** §2.4 (taxonomia completa da
  lacuna de testes) — não-crítica; substância coberta pelos 15 invariantes como alvo
  de cobertura.

## 6.5 Coerência de terminologia

- **`Outcome`** é o termo único em todo o corpus; **zero** ocorrência de "resultado
  da classificação" (sinônimo do v1 descartado).
- Identificadores em inglês: `triages`, `consents`, `CompleteTriage`, `domain_events`
  etc., consistentes nos ADRs.
- **Nit (não-bloqueante):** `adr/0008.md` usa "Triagem" (prosa PT, maiúscula) numa
  frase onde o resto do corpus usa `Triage`/`triagem`. Uma ocorrência, contexto de
  prosa — o identificador EN (`aborted_by_revocation`) está correto. Cosmético.

## 6.6 Coerência módulo ↔ ADR

- **Todo ADR governante citado por módulo existe no v2** (0001–0015). Verificação
  número-a-número (Etapa 5): nenhum número v1 sem traduzir; conferido que `0007`∉04/07,
  `0009`∉07, `0013`∉03 (os casos onde um v1 não-traduzido apontaria errado).
- **Todo módulo MVP tem governantes coerentes com o de-para:** 01→{0005,0007,0013},
  02→{0005,0008}, 03→{0009}, 04→{0010}, 05→{0003,0010}, 06→{0011,0012,0013},
  07→{0003,0013,0014}. O mapa reverso ADR→módulo no `modulos/README.md` bate.
- **Incoerência reportada (não-bloqueante, decisão do autor pendente):** o módulo 03
  descreve o workflow de protocolo como `draft → in_review → published → retired`,
  enquanto o ADR 0009 define `draft → active → retired`. É **discrepância
  pré-existente do v1** (módulo 03 vs ADR 0016), preservada — **não** é invariante
  perdida nem regressão introduzida pela refundação. Requer alinhamento (qual lado
  ganha), mas não bloqueia o Movimento 2.

## 6.7 F-IDs intactos

- **Contagem v1 = v2: 89 = 89 F-IDs únicos**, conjunto **idêntico** (diff vazio).
- Nenhuma funcionalidade perdida, criada ou realocada entre módulos pela refundação
  de ADR. (Esperado: a refundação toca decisões, não unidades de trabalho.)

---

## Veredito

**O Movimento 1 está completo e íntegro? — SIM.**

Os três checks duros passam sem exceção:
- **0 referência órfã** (6.1) — a ponte v1→v2 é completa.
- **0 invariante perdida** (6.2) — nenhuma decisão de segurança clínica, LGPD ou
  isolamento desapareceu na consolidação 26→15.
- **0 contrato perdido** (6.3) — todos no estado final.

Pendências **não-bloqueantes** (documentais, decisão do autor — não invariante, não
contrato):
1. Estados do workflow de protocolo: módulo 03 (`in_review/published`) × ADR 0009
   (`active`). Discrepância herdada do v1; alinhar quando conveniente.
2. Nit de terminologia: "Triagem" em prosa no ADR 0008.
3. Brief do dashboard não migrado (default aplicado) — migrar como referência ou
   manter arquivado.
4. Prompt do relatório semanal (`adr/prompts/`) — artefato v1, reconciliar junto do
   tooling de drift / Movimento 2.

**O Movimento 2 (apps/packages → multi-repo) pode começar? — SIM.**

Razão: o corpus v2 é a fonte da verdade estável e auditada que o Movimento 2 exige
para a Etapa 7 (inventário código × corpus). As pendências acima são refinamentos
documentais que não afetam o alinhamento de código — nenhuma muda uma invariante,
um contrato ou a topologia. Recomenda-se resolver (1) antes de implementar o módulo
03 (Triagem), para o código não herdar a ambiguidade de nomenclatura de estado.
