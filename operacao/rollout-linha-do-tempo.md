# Rollout — linha do tempo do paciente (módulo 20)

Runbook do operador para publicar o módulo 20 (ADR 0035), ligar a linha do
tempo do profissional e do cidadão numa cidade e, no modo integrado, expor a
página para a tela de "sistemas externos" do e-SUS PEC. Bases:
[`rollout-prontuario.md`](rollout-prontuario.md) (19a),
[`rollout-modo-de-prontuario.md`](rollout-modo-de-prontuario.md) e
[`rollout-receita-controlada.md`](rollout-receita-controlada.md) (19d).

O módulo 20 monta a linha do tempo da pessoa na leitura, a partir dos
fornecedores de cada módulo (sem tabela nova de eventos), com três entradas: o
profissional no dashboard, o cidadão no wpda e o PEC por iframe. Há dois
interruptores, ambos desligados por padrão: `patient_timeline` e
`pec_external_view`.

**O PEC real ainda não foi provado.** O módulo foi entregue e provado contra o
simulador do PEC. Nenhuma cidade real liga `pec_external_view` antes dos gates
do §8 (rotasaude/api#64). O `patient_timeline` não depende deles.

## 1. Pré-requisitos

- Módulo 19a publicado e, para o resumo clínico e os documentos do cidadão, os
  documentos do 19c.
- Para `pec_external_view`: cidade em `record_mode: integrated` (módulo 16), com
  `patient_timeline` ligado, e origem e chave HMAC do PEC cadastradas (§4).
- Profissionais com CNES do estabelecimento e CPF ou CNS cadastrados, porque o
  PEC identifica o profissional por documento e CNES.

## 2. Publicação

Ordem obrigatória: **api → dashboard e wpda → maintenance**. Nenhum contrato
novo (`contracts` não mudou).

1. `api` (`9c7cec8` ou posterior), com **imagem nova**. Antes de cortar o
   tráfego:
   1. publique a imagem;
   2. faça backup dos bancos das cidades e pare o worker;
   3. rode `bin/rails city:migrate:all` **da imagem nova**. A migração
      `20261011800002_add_patient_timeline` cria os índices por pessoa e data,
      `clinical_record_openings.citizen_id` (com CHECK de exatamente um entre
      `patient_id` e `citizen_id`), `pec_settings` (`pec_origin` e `hmac_key`
      cifrada por cidade) e `pec_signature_uses`. É **irreversível**. Nunca
      migre fora do rake: a cidade trava em 503. Não há migração de
      plataforma;
   4. religue o worker.
2. `dashboard` (`b4640de` ou posterior) e `wpda` (`8401032` ou posterior),
   depois do api. Os dois são independentes entre si.
3. `maintenance` (`9127bb4` ou posterior), depois do api: o quadro do PEC e as
   mutations dependem dos campos novos.

Publicar não muda nenhuma cidade: os interruptores nascem desligados.

## 3. Interruptores por cidade

| Interruptor | Pré-requisito | Motivos de bloqueio |
|---|---|---|
| `patient_timeline` | nenhum | — |
| `pec_external_view` | `patient_timeline`, `record_mode: integrated`, `pec_origin` e `pec_hmac_key` | `patient_timeline_disabled`, `record_mode_not_integrated`, `pec_origin_missing`, `pec_hmac_key_missing` |

- Em dev e staging, ligue pelo `maintenance` (aba Funcionalidades).
- Em produção, que não tem maintenance, pelo console do Rails, como no 19c:
  `Platform::Features.set!(city:, key: "patient_timeline", enabled: true, maintainer:)`.
  A troca vai para a trilha da plataforma (`city.feature_changed`).
- Ligue `patient_timeline` primeiro e confira o §6 antes de pensar no
  `pec_external_view`.
- Com `patient_timeline` desligado, o dashboard e o wpda voltam ao
  comportamento do 19d (sem aba, sem botão, sem "Meu histórico").

## 4. Configurar o PEC (cidade `integrated`)

Origem (`pec_origin`) e chave HMAC (`pec_hmac_key`) são por cidade.

- **Pelo maintenance** (dev e staging): quadro "Página do PEC (sistemas
  externos)". A origem é normalizada ao salvar (`setCityPecOrigin`, anulável
  para limpar). A chave é trocada em dois cliques mais o TOTP do mantenedor
  (`rotateCityPecHmacKey`) e **nunca** é exibida de volta.
- **Pelo console do Rails** (produção, sem maintenance; ainda não há caminho
  sem console, rotasaude/api#65):

  ```ruby
  CityConnection.with(city) do
    PecExternal::Configure.set_origin(origin: "https://pec.<cidade>.gov.br")
    PecExternal::Configure.rotate_key(key: "<chave>")
  end
  PecExternal::Configure.audit!
  ```

  O `audit!` registra a troca (`maintenance.city.pec_config_changed`, só com o
  campo alterado, nunca o valor) e emite o evento `city.pec_config_changed`.
- A mesma chave vai no cadastro do PEC (§5). Combine a entrega por canal fora
  de chat e de ticket; para a custódia, veja
  [`custodia-de-chave.md`](custodia-de-chave.md).

## 5. URL a cadastrar no PEC

No PEC da cidade, em "sistemas externos", cadastre o sistema com **Local de
acesso "Atendimento"** e a URL:

```
https://<cidade>/pec/timeline?documento-profissional=[documentoProfissional]&documento-cidadao=[documentoCidadao]&cnes=[CNES]
```

`<cidade>` é o host do api da cidade. O PEC assina a URL com HMAC-SHA256 (da URL
até `&pec_sign=`) usando a chave do §4.

O que o api valida: janela de 5 min, assinatura de **uso único**, CNES, vínculo
do profissional com o estabelecimento e par validado do cidadão. A resposta leva
CSP `frame-ancestors <pec_origin>` e, em recusa, um 403 HTML com o código
(`pec_replayed`, `pec_expired`, `pec_professional_not_linked`,
`pec_citizen_missing` e os demais). O "carregar mais" usa
`GET /pec/timeline/more` com `X-Pec-View-Session` (15 min); sessão vencida dá
401 `pec_session_expired`. Limites por IP: 120/min e 300/min (429
`too_many_requests`).

## 6. Verificações pós-deploy

Com `patient_timeline` ligado numa cidade de teste:

1. **Profissional (`record`):** abrir o painel de um paciente e ver a aba "Linha
   do tempo". **Profissional (`integrated`):** botão "Linha do tempo" na fila e
   abertura justificada (abre só a linha do tempo). Sem vínculo ou sem abertura,
   a leitura é recusada.
2. **Cidadão:** "Meu histórico" no wpda. Par validado: resumo, documentos com
   PDF e código, linha do tempo. Par declarado: só triagens e agendamentos e o
   convite "Validar no posto". No `integrated`, o aviso de histórico parcial.
3. **PEC** (só com `pec_external_view`): abrir pelo PEC e ver a linha do tempo
   no iframe, com o aviso de parcial. Um segundo acesso com a mesma URL assinada
   deve dar `pec_replayed`.
4. **Relatório de aberturas:** o `municipal_admin` vê as aberturas, também no
   `integrated`.
5. **Logs:** a query de `/pec/*` aparece como `[FILTERED]`; nenhum CPF, CNS ou
   assinatura em log, evento ou erro.
6. **Interruptor desligado:** desligar `patient_timeline` devolve as telas ao
   19d.

Em dev, o simulador do PEC (só `development`) fica em
`http://pec-sim.localhost:3040/pec-simulator?city=maringa`, com a semente
`lib/timeline_crew.rb`.

## 7. Rollback

- A migração é aditiva: a **imagem anterior do api convive** com ela. Não
  desfaça a migração (irreversível).
- Para reverter o comportamento, desligue `pec_external_view` e depois
  `patient_timeline` por cidade; as telas voltam ao 19d.
- Se a chave vazou: troque pelo §4 e atualize o cadastro no PEC. Para parar o
  acesso do PEC de imediato, desligue `pec_external_view`.
- Em reversão de imagem, o dashboard, o wpda e o maintenance novos continuam
  funcionando com os interruptores desligados; reverta-os depois do api.

## 8. Gates de go-live

| Item | Bloqueia? |
|---|---|
| rotasaude/api#64 — proxy repassa `Host` e `X-Forwarded-Proto` em `/pec/*` | **Sim** (`pec_external_view`) |
| rotasaude/api#64 — access log filtra a query de `/pec/*` | **Sim** (`pec_external_view`) |
| rotasaude/api#64 — nenhum proxy injeta `X-Frame-Options` | **Sim** (`pec_external_view`) |
| rotasaude/api#64 — cadastro no PEC e chave entregue | **Sim** (`pec_external_view`) |
| rotasaude/api#64 — chamada e iframe provados num PEC real (5.5.x) | **Sim** (`pec_external_view`) |
| rotasaude/api#64 — CNES e CPF ou CNS dos profissionais cadastrados | **Sim** (`pec_external_view`) |
| rotasaude/api#64 — relógio do servidor e do PEC conferido (janela de 5 min) | **Sim** (`pec_external_view`) |
| rotasaude/api#64 — tetos do rate limit revistos | **Sim** (`pec_external_view`) |
| rotasaude/api#65 — origem e chave do PEC sem console em produção | Não (hoje por console) |
| Gates do 19a a 19d (api#53, api#56, api#57, api#62, api#63) para o resumo e os documentos do cidadão | Conforme o módulo de origem |

Pendências sem bloqueio: api#66 a #71 (veja
[`modulos/20--linha-do-tempo.md`](../modulos/20--linha-do-tempo.md)).

## ADRs relacionados

0017 (par validado), 0028 (interruptores e modos), 0031 (regras de leitura),
0035 (linha do tempo e página do PEC).
