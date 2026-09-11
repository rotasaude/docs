# Nota de revisão operacional

- Data: 2026-06-18
- Status: viva (atualizar a cada incidente ou auditoria operacional)

## Propósito

Esta nota não é ADR. Ela conecta as decisões dos ADRs aos **procedimentos** que o time precisa lembrar **na hora do incidente**. Cada item aponta para o(s) ADR(s) que governa(m) a decisão de design, e descreve o que fazer quando algo dá errado em produção.

## Inventário operacional

### Fila / jobs ([ADR-0001](0001.md), [ADR-0008](0008.md))

- Profundidade da fila `urgent` deve ficar em 0 em condição normal. Se > 0 por mais de 30s, investigar antes de qualquer outra ação.
- `bin/jobs` é supervisor único; reiniciar o papel `worker` via Kamal (`kamal app boot --roles worker`) **não** perde job — Solid Queue persiste em Postgres.
- Pool de threads em `config/queue.yml`. Mudança exige redeploy do papel worker.

### Eventos e replay ([ADR-0003](0003.md), [ADR-0005](0005.md), [ADR-0009](0009.md))

- Tabela `domain_events` é **imutável**. Único UPDATE legítimo é em `published_at`.
- Para reenfileirar consumidores: `bin/rails runner scripts/replay_domain_events.rb -- --dry-run` **sempre primeiro**.
- Tabela `processed_events` cresce monotônico — purga em `recurring.yml` (ADR-0007).

### Webhook do WhatsApp ([ADR-0010](0010.md), [ADR-0014](0014.md))

- HMAC validado antes de qualquer INSERT. Se `WHATSAPP_APP_SECRET` for rotacionado, a janela de validade dupla **não está implementada** — rotação é break.
- Ack < 1s é mandatório. Se a janela do controller estiver lenta, suspeitar de Postgres antes de tudo.
- `SendWhatsappJob` é separado por design (HTTP fora do lock). Não tente "otimizar" colocando HTTP dentro do consumer.

### Secrets ([ADR-0011](0011.md))

- Master key é **o pivot**. Perda = todos os dados criptografados ficam inacessíveis. Backup offline obrigatório.
- Em produção, `apps/api/deploy/production/secrets` exige `op` (1Password CLI) no host de deploy.
- Em dev, `.env.development.local` (ignorado pelo git) é o fallback aceito.

### Protocolos ([ADR-0013](0013.md), [ADR-0016](0016.md), [ADR-0017](0017.md))

- Definição ativa por `(name, municipality_id)` é única (índice parcial). Tentativa de ativar duas falha em transação — não tentar contornar via SQL direto.
- Lint offline (`scripts/validate_protocol_definition.rb`) **antes** de importar. Definição inválida não entra no banco — `Definition#before_save` valida.
- Cache de `Protocols.current` é por process. Mudança via SQL direto exige restart do app.

### Consentimento ([ADR-0012](0012.md))

- Mudança de versão de política = todos os cidadãos `consented` em V_n precisam re-consentir no próximo inbound. Pico operacional **previsível** — comunicar com antecedência.
- Revogação aborta triagem em curso (`status: :aborted_by_revocation`). Relatórios não contam essas triagens como "completadas".
- `Consents.interpret` é heurística — falsos positivos em "sair" são preferíveis a falsos positivos em "aceito".

### Relatórios e dashboard ([ADR-0007](0007.md))

- `ReportSnapshot` é imutável depois de criado. Para "corrigir" um relatório errado, expirar o token + gerar snapshot novo.
- `DashboardMetric` é reconstrutível. Suspeita de inconsistência? Rodar `scripts/rebuild_dashboard_metrics.rb` em janela controlada.
- HMAC do token usa `report_signing_key` em credentials — rotação ainda **não suportada** (mesma observação do WhatsApp APP_SECRET).

## Runbooks curtos

### "O cidadão não recebeu mensagem"

1. checar `OutboundMessage` por `to=phone`;
2. se status = 5xx ou nenhum registro, checar profundidade da fila `realtime`;
3. se fila vazia mas job não rodou, checar se `triagem.completed` apareceu em `domain_events`;
4. se evento existe mas `published_at IS NULL`, rodar replay seletivo ([ADR-0009](0009.md)).

### "A Meta voltou a entregar webhook depois de ack ok"

1. confirmar que o ack foi 2xx no log — se foi 5xx, é Postgres caído;
2. se ack foi 2xx mas Meta reenviou, é janela de retry agressiva — não fazer nada (a dedup ADR-0005 protege);
3. se a Meta desabilitou o webhook, reativar no painel da Meta — o HMAC continua válido.

### "Dashboard mostra número errado"

1. **não** mexer em `dashboard_metrics` à mão;
2. rodar `scripts/rebuild_dashboard_metrics.rb` em janela noturna ou manualmente em incidente;
3. se a inconsistência persistir, é bug no agregador — abrir issue, não tentar consertar via SQL.

### "Cidadão pediu 'esquecimento' por LGPD"

1. localizar `Conversation` por `phone`;
2. `RevokeConsent.call(conversation:, reason: "lgpd-art18")`;
3. agendar purga de `InboundMessage.raw` para `NULL` após janela de retenção configurada (rotina ainda não automatizada — manual por ora).

## Pontos cegos conhecidos (a fechar)

- rotação de `WHATSAPP_APP_SECRET` e `report_signing_key` sem janela de break — implementar suporte a dois secrets;
- purga programática de `InboundMessage.raw` ainda manual;
- `AlertMunicipalityJob` enfileira `DispatchMunicipalityAlertJob` que não existe ainda — definir canal por município.
