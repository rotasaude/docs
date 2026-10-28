# Módulo 25 — Vigilância epidemiológica

- **Estado:** Entregue (2026-10-28; F-25.1–F-25.8 `Verified`; `Fechado` depois de validar a lista e as janelas com a vigilância piloto e de uma notificação real digitada no Sinan)
- **Tipo:** Ciclo 2

## Escopo

Vigilância epidemiológica (ADR 0043): lista nacional de notificação compulsória
versionada na plataforma (Portaria 11.211/2026), com CID-10 e CIAP-2 por doença e
doenças locais da cidade; suspeita confirmada pelo médico ou enfermeiro na consulta,
sem bloquear; rede de segurança na finalização; fila da vigilância com notificação
manual, ficha, andamento até a digitação no Sinan com o número oficial; prazos
(imediata 24 h, semanal 7 dias), alertas sem dado do paciente e notificação negativa
semanal; ficha em PDF, CSV por semana e relatório sem número; painel por bairro e
semana epidemiológica com supressão.

**Fora, por enquanto:** envio automático ao Sinan; vacinas (29); DO/DN; GAL;
investigação de campo (35).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0043 | Lista, suspeita, rede de segurança, fila, prazos, saídas, painel |
| 0031 | Consulta e lista de problemas |
| 0025 | Supressão de números pequenos |
| 0042 | Paciente sem CPF e bairro do domicílio |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-25.1 | Interruptor, papel de vigilância e lista de doenças | api, dashboard, maintenance | 0043 |
| F-25.2 | Sugestão de notificação na consulta | api, dashboard | 0043, 0031 |
| F-25.3 | Rede de segurança na finalização | api | 0043 |
| F-25.4 | Fila, notificação manual e ficha | api, dashboard | 0043, 0042 |
| F-25.5 | Andamento, encerramento e número oficial | api, dashboard | 0043 |
| F-25.6 | Prazos, alertas e notificação negativa | api, dashboard | 0043 |
| F-25.7 | Ficha em PDF, CSV por semana e relatório sem número | api, dashboard | 0043 |
| F-25.8 | Painel por bairro e semana epidemiológica | api, dashboard | 0043, 0025 |

## Critério de fechamento

- F-25.1 a F-25.8 verificadas, com prova no navegador.

## Entrega (2026-10-28)

| Repo | Commit | Observação |
|---|---|---|
| `api` | `139ab0e7` | 32 commits; suíte completa 7615/0; migrações de plataforma `20261027900001` e de cidade `20261027900002` |
| `dashboard` | `956f57f` | 18 commits; 4368/0 |
| `maintenance` | `83239db` | interruptor `surveillance` e lista de doenças da plataforma (publicação com step-up) |
| `catalog` | pendente | Task 6 do maintenance, depois da autorização |
| `docs` | contrato §14 (ficha) e §15 K1–K17 | runbook [`operacao/rollout-vigilancia.md`](../operacao/rollout-vigilancia.md) |

**Prova técnica:** Portaria 11.211/2026 e 58 fontes oficiais vendorizadas com SHA-256 no api; 71 doenças cobrindo os 66 itens, com fonte por linha; CID-10 e CIAP-2 conferidos; ficha de notificação individual do Sinan publicada como §14; semana epidemiológica conferida no calendário do Sinan (SE 53/2025).

**Prova no navegador (2026-10-28):** sugestão na consulta sem bloquear; rede de segurança; fila com prazos e imediatas; notificação manual; ficha, andamento, número oficial e descarte; PDF e CSV; notificação negativa; painel com supressão e sinal; doenças locais; interruptor e publicação da lista no maintenance (step-up, TOTP errado, concorrência, auditoria).

**Decisões tomadas na execução** (contrato §15): semana 1 = a que contém 4 de janeiro; códigos genéricos (F99, L98.9, C80, Z20.9) e SRAG só manuais; janelas de repetição editáveis (sífilis congênita 36500, gestante com HIV 270) a validar com a vigilância; sinal calculado sem reconstruir célula suprimida (célula 1–4 conta 2,5); a API aceita notificar consulta finalizada; analista vê a lista de doenças.

**Gates:** validar a lista, os códigos genéricos e as janelas com a vigilância da cidade piloto; primeira notificação real digitada no Sinan.

## Integrações externas

Sinan / e-SUS Sinan / e-SUS Notifica sem API para terceiros (digitação); lista nacional versionada (Portaria 11.211/2026). Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 14, 19 e 24a.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-27 — Desenho aprovado e `Planejado` (ADR 0043, spec
  `superpowers/specs/2026-10-27-module-25-disease-surveillance-design.md`).
- 2026-10-28 — **Implementado, publicado e Entregue** (ver "Entrega").
