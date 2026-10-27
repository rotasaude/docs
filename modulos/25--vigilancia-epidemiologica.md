# Módulo 25 — Vigilância epidemiológica

- **Estado:** Planejado
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
