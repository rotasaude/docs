# Decisão — Lifecycle de protocolo: `published` ≠ `active` (ponteiro por cidade)

> Resolve a pendência (1) do veredito do Movimento 1: a discrepância entre o
> workflow de autoria do módulo 03 (`draft/in_review/published/retired`) e o
> estado `active` do ADR de Protocol engine. Decisão tomada pelo autor;
> esta nota especifica **o que muda no corpus v2** para o agente aplicar com
> precisão, sem reinterpretar. Não é prompt de execução — é especificação de
> correção.

## Decisão (fixada)

A vida de um protocolo tem **dois eixos independentes**, antes confundidos
num só:

### Eixo 1 — Lifecycle da definição (global, por versão)

```
draft → in_review → published → retired
```

- `draft` — em escrita; mutável; não pode ser ativada.
- `in_review` — submetida aos quatro olhos; não pode ser ativada.
- `published` — aprovada pelos quatro olhos; **imutável**; disponível para
  ativação. **NÃO significa "em uso".**
- `retired` — aposentada; não pode mais ser ativada (regra abaixo).

### Eixo 2 — Vigência por cidade (ponteiro)

- Cada município aponta para **exatamente uma** versão `published` como sua
  versão `active`.
- Cidade A pode estar `active` na v3 enquanto cidade B está na v2 do mesmo
  protocolo. Versões diferentes por cidade são realidade do piloto.
- A versão `active` de uma cidade é a que **novas** triagens daquela cidade
  consomem.

## Quatro regras que decorrem (todas devem entrar no ADR)

### R1 — Só `published` pode ser ativada
O ponteiro `active` de uma cidade só aponta para versão em estado
`published`. Tentar ativar `draft`, `in_review` ou `retired` falha.

### R2 — `ActivateProtocolVersion` é comando próprio
Distinto de `PublishProtocol`:
- **`PublishProtocol`** — global; move a versão `in_review → published`;
  exige role `protocol_publisher` + step-up MFA (já decidido).
- **`ActivateProtocolVersion`** — por cidade; aponta o ponteiro `active` da
  cidade para uma versão `published`; ato deliberado e auditável.

São dois atos, duas entradas de auditoria (`protocol.published` e
`protocol.activated`), possivelmente dois níveis de permissão (publicar é da
plataforma/autoria; ativar pode ser do `municipal_admin` da cidade — decidir
na aplicação, mas o comando é separado).

### R3 — Triagem congela a versão no início
Uma triagem iniciada sob a versão `active` N **termina em N**, mesmo que a
cidade ative N+1 no meio do fluxo. Isto já é garantido pelo snapshot por
`protocol_version` (ADR de CQRS/snapshots). **O ADR de Protocol engine deve
cruzar essa dependência explicitamente** — a separação `published`/`active`
só é segura por causa do congelamento por snapshot. Sem essa amarração, a
nota fica incompleta.

### R4 — `retire` exige que nenhuma cidade esteja `active` na versão
Aposentar uma versão **falha** se qualquer cidade ainda a tem como `active`.
Forçar migração explícita antes de aposentar. Nunca trocar a versão vigente
de uma cidade por baixo dos panos (perigoso em contexto clínico).
- Comando `RetireProtocolVersion` valida: `count(cidades active nesta versão) == 0`.
- Se houver cidade ativa, retorna falha com a lista de cidades a migrar.

## O que muda no corpus v2 — instruções de aplicação

### No ADR de Protocol engine (0010 no esquema novo)

1. **Adicionar a distinção dos dois eixos** na seção de Decisão: lifecycle
   da definição (estados) vs. ponteiro de vigência por cidade.
2. **Adicionar `ActivateProtocolVersion`** como comando, ao lado de
   `PublishProtocol` e `RetireProtocolVersion`. Especificar assinatura e
   invariante (R1, R2).
3. **Adicionar a regra R4** ao `RetireProtocolVersion`.
4. **Cruzar R3 com o ADR de CQRS/snapshots** — referência explícita de que
   a triagem congela a versão e por que isso torna a ativação segura.
5. **Definir o contrato do ponteiro** (ver modelo de dados abaixo).

### No modelo de dados (estado final, EN)

A versão ativa por cidade é um **ponteiro**, não um estado na própria versão
(porque "active" não é global — varia por cidade). Modelo recomendado:

```
protocol_definitions          # a versão e seu lifecycle global
  id
  protocol_key                # qual protocolo (ex.: "dengue")
  version                     # número da versão
  status                      # draft | in_review | published | retired
  ... (definição, autoria, etc.)

active_protocol_versions      # o ponteiro: qual versão cada cidade usa
  municipality_id             # FK
  protocol_key                # qual protocolo
  protocol_definition_id      # FK → a versão published ativa
  activated_at
  activated_by
  UNIQUE(municipality_id, protocol_key)   # uma ativa por (cidade, protocolo)
```

`Protocols.current(municipality_id, protocol_key)` resolve via
`active_protocol_versions`, não via `status`. **Atenção EN:** nomes em
inglês (convenção fixa do v2) — `active_protocol_versions`, não
`versoes_ativas`.

### Invariantes a adicionar à lista de invariantes do ADR

- **INV-protocol-1:** o ponteiro `active` de uma cidade sempre aponta para
  uma versão em estado `published` (nunca draft/in_review/retired).
- **INV-protocol-2:** exatamente uma versão ativa por (cidade, protocolo) —
  garantido por unique constraint.
- **INV-protocol-3:** triagem termina na versão em que começou
  (congelamento por snapshot); ativar nova versão não afeta triagens em voo.
- **INV-protocol-4:** versão `retired` não tem nenhuma cidade ativa nela.

Estas quatro entram na suíte de invariante do módulo 03 (camada de teste do
ciclo de desenvolvimento), e na auditoria de qualquer reescrita futura.

### No módulo 03 (`docs/modulos/03--*.md` no v2)

1. Atualizar a descrição do workflow para os dois eixos.
2. Adicionar funcionalidade nova: ativação de versão por cidade
   (`ActivateProtocolVersion`). Isso é uma F-ID nova — **atenção:** muda a
   contagem de 89. Registrar como F-03.18 (ou o próximo número livre do
   módulo) e anotar que a contagem de funcionalidades do MVP subiu.
3. Adicionar funcionalidade de aposentadoria com a guarda R4.
4. Atualizar o critério de fechamento do módulo com as 4 invariantes.

### Eventos de domínio afetados (EN)

- `protocol.published` — já existe; é global.
- `protocol.activated` — **novo**; por cidade. Payload:
  `{municipality_id, protocol_key, protocol_definition_id, version, activated_by}`.
- `protocol.retired` — já existe ou novo; global, com a guarda R4.

Estes contratos de evento vão para `contracts/events/` (repo de contratos,
ADR de contracts) no estado final.

## Impacto na contagem de funcionalidades

O veredito registrou 89 F-IDs. Esta decisão adiciona pelo menos 2
funcionalidades ao módulo 03 (ativação por cidade; aposentadoria com guarda).
A contagem do MVP passa a ~91. O seeder de funcionalidades (quando rodar)
deve refletir o número real após esta correção, não o 89 antigo.

## O que NÃO muda

- Os quatro olhos (`draft → in_review → published`) permanecem; a decisão
  não toca o fluxo de autoria, só separa "aprovado" de "vigente".
- O step-up MFA na publicação permanece.
- O snapshot por versão (CQRS) permanece — é o que torna tudo seguro.
- A imutabilidade de versão `published` permanece.

## Ordem de aplicação

Esta correção deve ser aplicada ao corpus v2 **antes** da Etapa 7
(inventário do Movimento 2). Razão: a Etapa 7 vai inventariar o código real
contra o corpus v2; se o corpus ainda tem a ambiguidade, o inventário herda
a discrepância que esta decisão resolve. Aplicar agora = o código é medido
contra o modelo correto.

Sugestão: uma micro-etapa (chamável "Etapa 6.5") que aplica esta nota ao
ADR de Protocol engine, ao módulo 03, e aos contratos de evento no v2, com
o mesmo rigor das etapas anteriores (read da fonte, write no destino,
validação de que as 4 invariantes entraram e a contagem de F-IDs foi
atualizada).
