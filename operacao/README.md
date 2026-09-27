# Operação — Rota Saúde

Runbooks operacionais que sustentam decisões dos ADRs sem virar ADR em si.
Cada arquivo aqui descreve **como** um procedimento é executado em produção;
o **por quê** vive em `docs/adr/`.

Documento vivo. Diferente dos ADRs (imutáveis), runbooks podem ser editados —
a história está no git.

## Runbooks

| Arquivo | Escopo | ADRs relacionados | Estado |
|---|---|---|---|
| `rollout-agendamento.md` | Ordem de deploy e rollback do subprojeto 4 (migração irreversível de horários) | 0019, 0020 | Vigente |
| `provisionar-canal-whatsapp.md` | Registrar, validar e rotacionar o canal WhatsApp de uma cidade | 0007, 0013, 0017, 0020 | Vigente |
| `custodia-de-chave.md` | Onde a chave do AR Encryption vive, como rotaciona, quem acessa | 0013, 0020 | Vigente |
| `atender-requisicao-lgpd.md` | Acesso, revogação, eliminação (sem decisão) e revisão de decisão automatizada | 0013, 0014, 0017, 0020 | Vigente |
| (pendente) `monitoramento-fila.md` | Saúde de Solid Queue: lag, `failed_executions`, jobs travados | 0001, 0006 | A produzir |
| (pendente) `restore-postgres.md` | RPO/RTO, drill de restore, atenção à fila no mesmo banco | 0001 | A produzir |
| (pendente) `plantao-alerta-urgente.md` | Quem recebe, cadeia de escalonamento, janela de 24h do WhatsApp | 0006, 0013 | A produzir |
| (pendente) `provisionamento-municipio.md` | Comando, secrets a configurar, validação pós-provisão | 0013 | A produzir |
| (pendente) `publicacao-protocolo.md` | Quatro olhos, step-up MFA, fallback de backstop | 0009, 0011, 0012 | A produzir |
| (pendente) `migracao-multi-repo.md` | Extração de cada app para repo próprio, `contracts`, `docs` | 0002 | A produzir |

## Convenções

- Nome de arquivo: kebab-case, ação no infinitivo ou substantivo concreto.
- Cabeçalho fixo de cada runbook:
  - **Quando aplicar**
  - **Pré-requisitos**
  - **Passos** (numerados)
  - **Validação** (como saber que deu certo)
  - **Rollback** (se aplicável)
  - **ADRs relacionados**
- Runbook **não decide**. Se um passo exige decisão arquitetural, abre ADR.
- Mudança em runbook não precisa de cerimônia: PR direto. Mudança que altera
  o que o ADR-pai exige precisa virar ADR novo.

## O que NÃO vive aqui

- Decisões arquiteturais — vão para `docs/adr/`.
- Pulsos semanais do drift report — vão para `docs/relatorios/`.
- Planos de implementação por agente — vão para `docs/superpowers/plans/`.
- Mapa de módulo e funcionalidade — vão para `docs/modulos/`.

> **Nota de proveniência.** O corpus v1 mantinha uma "nota de revisão
> operacional" viva (`nota-revisao-operacional.md`) que conectava ADRs a
> procedimentos de incidente, com seções numeradas (`§1.1`, `§5.4`…). Esse
> documento ficou no arquivo histórico v1. O conteúdo de incidente que ele
> reunia se redistribui nos runbooks acima (a produzir) e nos "Em aberto" dos
> ADRs v2; as referências `§X.Y` que outros docs faziam a ele foram
> reapontadas para o ADR v2 correspondente.
