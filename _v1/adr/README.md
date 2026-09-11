# Decisões de Arquitetura — Rota Saúde

Registros de decisão (ADRs) do piloto municipal. Cada arquivo documenta **uma**
decisão, seu contexto e suas consequências. Decisões aceitas são imutáveis;
mudanças entram como ADRs novos que emendam.

Trabalho organizado em torno destes ADRs vive em `docs/modulos/` (mapa de
módulos) e segue o ciclo descrito em `docs/ciclo-desenvolvimento.md`.

## Índice

| # | Item | Status |
|---|------|--------|
| 0001 | [Transporte: Solid Queue](0001.md) | Aceita |
| 0002 | [Estrutura de aplicações](0002.md) | Aceita (emendado por 0018) |
| 0003 | [Pub/sub de domínio (DomainEvents)](0003.md) | Aceita (emendado por 0020) |
| 0004 | [Atomicidade write + enqueue](0004.md) | Aceita |
| 0005 | [Idempotência dos consumers](0005.md) | Aceita (emendado por 0020) |
| 0006 | [Camada de commands](0006.md) | Aceita |
| 0007 | [CQRS: projeções e snapshots](0007.md) | Aceita |
| 0008 | [Filas com prioridade (segurança clínica)](0008.md) | Aceita |
| 0009 | [Auditoria LGPD via domain_events](0009.md) | Aceita (emendado por 0023) |
| 0010 | [Borda de ingestão: ack rápido e idempotência](0010.md) | Aceita (emendado por 0014, 0021) |
| 0011 | [Payload bruto: criptografia e retenção](0011.md) | Aceita (emendado por 0021, 0024) |
| 0012 | [Máquina de estados da conversa e consentimento](0012.md) | Aceita (emendado por 0021) |
| 0013 | [Motor de protocolos: fluxo e ramificação](0013.md) | Aceita (emendado por 0016) |
| 0014 | [Resposta ao cidadão fora do lock](0014.md) | Aceita |
| 0015 | [Classificação: prioridade e explicabilidade](0015.md) | Aceita (scoring fechado por 0017) |
| 0016 | [Cadastro de protocolos: storage, workflow e RBAC](0016.md) | Aceita |
| 0017 | [Modelos de scoring: weighted e decision_table](0017.md) | Aceita |
| 0018 | [Topologia física do monorepo: 4 apps](0018.md) | Aceita (layout superseded por 0025; papéis de app permanecem) |
| 0019 | [Isolamento RLS multi-tenant](0019.md) | Aceita |
| 0020 | [Tenant em eventos e jobs](0020.md) | Aceita |
| 0021 | [Roteamento de canal e ingestão multi-tenant](0021.md) | Aceita |
| 0022 | [Autenticação, sessão e MFA](0022.md) | Aceita |
| 0023 | [RBAC: memberships e tier de plataforma](0023.md) | Aceita |
| 0024 | [Provisionamento, custódia de secrets e config por cidade](0024.md) | Aceita |
| 0025 | [Topologia multi-repo: um repositório por aplicação](0025.md) | Aceita (emenda 0018) |

## Documentos auxiliares

- [`nota-revisao-operacional.md`](nota-revisao-operacional.md) — documento vivo
  que conecta ADRs a procedimentos operacionais; atualizado por incidente.
- [`prompts/`](prompts/) — prompts de execução vinculados ao ciclo
  (atualmente: relatório semanal de drift).

## Itens em aberto

Itens reconhecidos como **não decididos** que afetam o sistema. Cada um vira
ADR próprio quando entrar no ciclo.

### Arquitetura

- **UI de autoria de protocolos** — JSON editor + preview ao vivo no MVP;
  construtor gráfico fora de escopo. ADR pendente (toca 0016).
- **Contrato de tipos Ruby ↔ TypeScript** (`packages/types`) — 0013 endereçou
  a definição de protocolo; o caveat do 0002 segue em aberto para o resto.
- **Design system compartilhado** (`packages/ui`) — três frontends do 0018
  exigem componentes compartilhados; ADR pendente.
- **Brief formal do dashboard da cidade** — referenciado por
  `docs/modulos/05--dashboard.md` como governante; existe como dívida
  documental, não como arquivo formal no repo.

### Confiabilidade e segurança clínica

- **SLA do caminho de priorização clínica** (0008, referenciado por 0015) —
  precisa de input clínico/operacional do município.
- **Mecanismo de alerta urgente** (SLA + escalonamento + monitoramento +
  plantão) — amarra 0008, §2.2 da nota, §7.2 da nota. ADR(s) próprio(s).
- **Idempotência HTTP em consumers de efeito externo** (§1.2 da nota) —
  `NotifyCitizenJob`, `AlertMunicipalityJob`: marcador antes do efeito
  externo produz at-most-once silencioso. Decisão à parte do 0005/0020.

### Operação e custódia

- **Chave por tenant** (envelope encryption via KMS) — endurecimento
  pós-piloto da custódia do 0024.
- **Suspensão/desprovisionamento de cidade** — inverso do
  `ProvisionMunicipality` (0024): retenção pós-saída, base legal.
- **Monitoramento de Solid Queue** (§2.2 da nota) — lag, jobs travados,
  `failed_executions` órfãos. ADR próprio.
- **Backup/restore do Postgres** (§2.3 da nota) — RPO/RTO, drill de restore;
  fila vive no mesmo banco do estado.

### Identidade e LGPD

- **Integração gov.br OIDC** (seam do 0022 está pronto; ativação pendente).
- **Recovery de MFA** (fluxo de perda de dispositivo além dos recovery codes).
- **Imutabilidade vs. Art. 18 LGPD** (§5.1 da nota) — o que é apagável vs.
  retido por base legal; como "esquecer um cidadão" coexiste com snapshots
  que permanecem por versionamento de protocolo.
- **Art. 20 LGPD** (§5.2) — revisão humana de decisão automatizada: quem
  revisa, quando, em que canal.
- **Assinante real de `consent.revoked`** (§5.3) — hoje publicado sem
  consumer; base legal de retenção pós-revogação não declarada.
- **Reconciliação de identidade do cidadão com CNS/SUS** (§2.5 da nota) —
  pré-condição para `wpda` ir além do MVP.
