# Rollout — modo de prontuário e exportação LEDI (módulo 16)

Runbook do operador para publicar o módulo 16 (ADR 0028) e configurar uma
cidade. Ambiente de prova local do PEC: [`pec-local-dev.md`](pec-local-dev.md).

## 1. Publicação

Ordem obrigatória:

1. `contracts` com a tag `session-v1.1.0` (`features` na sessão).
2. `api`, numa imagem com as gems `thrift`, `rubyzip` e `csv`. Antes de cortar o tráfego:
   - `bin/rails db:migrate` na plataforma, que cria `city_features`, os campos de cidade, as terminologias, os retratos do CNES e `city_production_summaries`;
   - `bin/rails platform:triggers`;
   - `bin/rails city:migrate:all`, com as migrações `20261005200001` (credenciais, equipes e CNES/CPF/CNS) e `20261006200001` (`ledi_outbox`).
   Nunca migre uma cidade fora do rake.
3. `maintenance`, `admin` e `dashboard`. Qualquer um deles contra um `api` antigo quebra a tela.

Variáveis de ambiente:

- `config.x.cadsus_pdq_url` só existe em produção; staging usa a homologação do CADSUS.
- `LEDI_PEC_CA_FILE` só vale em development e test. Em produção, o PEC da cidade precisa de um certificado de CA pública.

Tudo nasce desligado: `record_mode=off` e interruptores `off`. Publicar o
módulo não muda o comportamento de nenhuma cidade.

## 2. Terminologias (plataforma, uma vez por versão)

```bash
bin/rails 'terminology:import[cid10,<versão>,<zip ou pasta>]'
bin/rails 'terminology:import[ciap2,<versão>,<zip ou pasta>]'
bin/rails 'terminology:import[sigtap,AAAAMM,<zip ou pasta>]'
```

A SIGTAP é mensal. A partir do dia 5, se a competência corrente não foi
importada, o console mostra o alerta (`sigtap_alert`).

## 3. CNES (plataforma, mensal)

```bash
bin/rails 'cnes:import[AAAAMM,BASE_DE_DADOS_CNES_AAAAMM.ZIP]'
```

- A importação é filtrada pelo IBGE das cidades ativas.
- A saída nunca traz CPF nem CNS.
- Cidade sem IBGE é pulada e aparece listada.
- Nada é aplicado ao cadastro da cidade: ela confirma as propostas.

## 4. Configurar uma cidade

1. **Operador**, no console (`admin`), em Cidades → Ficha:
   - código IBGE;
   - modo (`integrated` ou `record`, com confirmação);
   - endereço do PEC (`https://`, sem usuário nem senha).
2. **`municipal_admin`**, no painel da cidade (`dashboard`):
   - em e-SUS → Integrações, cadastra a credencial do e-SUS PEC (step-up) e testa a conexão;
   - faz o mesmo com a credencial do CADSUS, se for usar a consulta.
3. **`municipal_admin`**, em e-SUS → CNES: confirma as propostas de vínculo de unidades, equipes e profissionais.
4. **Mantenedor**, no `maintenance`: liga `ledi_export` e/ou `cadsus_lookup`. A tela mostra o que falta para cada um funcionar. O `maintenance` só existe em dev e staging: em produção os interruptores ficam desligados até um ADR próprio.

## 5. Gates antes de ligar `ledi_export` numa cidade real

| Card | Bloqueia? |
|---|---|
| rotasaude/api#43 — LGPD: `last_error` e `payload` da `ledi_outbox` fora da máscara e da exclusão | **Sim** (Crítica) |
| rotasaude/api#41 — observar o aceite (2xx) e o reenvio depois do aceite num PEC com CNES real | **Sim** |
| rotasaude/api#42 — fichas `failed` sem saída | Alta |
| rotasaude/api#44 — teto de tempo por execução do `DeliverJob` | Média |

## 6. Acompanhar

- **Console:** Produção das cidades mostra as competências corrente e anterior, com prazo pela tabela oficial do SIAPS (`config/ledi/siaps_deadlines.yml`) e alertas `attention`/`critical`.
- **Cidade:** e-SUS → Produção e-SUS mostra as fichas, os motivos de recusa e o reenvio (step-up).
- **Credencial recusada pelo PEC:** a exportação da cidade pausa e o aviso aparece em Integrações.
